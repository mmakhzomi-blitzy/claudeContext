# Consumer-Side Runner Ops Migration Guide

> **Related docs**:
> - `RUNNER_OPS_FRAMEWORK.md` — operation registration, session lifecycle, error handling
> - `ARCHITECTURE_REFERENCE.md` — system overview (BlitzyClient SDK -> Mediator -> Worker)
> - `CODE_DOWNLOADER_MIGRATION.md` — detailed code-downloader 3-phase example
> - `TASKS.md` — master task checklist

---

## Overview

Each consumer job currently runs locally: it clones the repo via `download_repository_to_disk()`,
creates a `BashSession` subprocess, and executes commands. The runner ops migration replaces
this with remote execution on a Kubernetes worker pod via the `RunnerSession` API.

**What changes**:
- `download_repository_to_disk()` — skipped (worker clones on startup)
- `restart_bash_session()` — replaced by `runner_session.restart_session()` (sync)
- `handle_bash_tool_response()` — replaced by `runner_session.run_bash(command)` (sync)

**What stays local**: LLM calls, Neo4j graph operations, GCS storage, Pub/Sub notifications.

**Flag**: `USE_RUNNER` env var, checked via `should_use_runner()`. When `false` (default),
behavior is identical to pre-migration.

---

## Common Pattern

Every consumer migration follows the same 3-step structure:

### Step 1 — RunnerSession lifecycle in `main.py`

```python
from blitzy_utils.blitzy_client import BlitzyClient
from blitzy_utils.runner_ops.client import should_use_runner
from blitzy_utils.runner_ops.session import RunnerSession

runner_session = None
if should_use_runner():
    client = BlitzyClient()
    runner_session = RunnerSession(client, company_id, job_id)
    runner_session.start()  # creates runner or reuses existing, waits for READY

try:
    helper = SomeHelper(..., runner_session=runner_session)
    # ... existing graph/processing logic ...
finally:
    if runner_session:
        runner_session.stop()  # tears down only if this session created the runner
```

### Step 2 — Helper accepts `runner_session`

```python
class SomeHelper:
    def __init__(self, ..., runner_session=None):
        self.runner_session = runner_session
        self.bash_session = None  # kept for local fallback path
```

### Step 3 — Flag-branch call sites

Use `self.runner_session is not None` as the discriminator:

```python
# setup() — skip clone + local bash init
if self.runner_session is None:
    download_repository_to_disk(...)
    self.bash_session, _ = await restart_bash_session(...)
else:
    logger.info("RunnerSession active — skipping local repo clone and bash session init")

# LLM bash tool handling — replace async calls with sync runner calls
if restart:
    if self.runner_session is not None:
        self.runner_session.restart_session()       # sync, no await
    else:
        self.bash_session, _ = await restart_bash_session(...)
    tool_result = "Bash session restarted successfully."
else:
    if self.runner_session is not None:
        tool_result = self.runner_session.run_bash(command)  # sync, no await
    else:
        tool_result = await handle_bash_tool_response(command=command, session=self.bash_session)
```

**Key detail**: `run_bash()` and `restart_session()` are **sync** (they poll internally).
The original `handle_bash_tool_response()` and `restart_bash_session()` are **async**.
In the runner branch, `await` is simply dropped.

---

## Tool-Level Runner Routing

LLM agents call tools via `process_tool_call()` which dispatches to tool implementations in
`archie-shared`. Some tools access the local filesystem — these need runner-awareness so they
route to the worker pod when `runner_session` is active.

### Which tools need routing?

Only **2 tools** touch the local filesystem:

| Tool | Module | Local implementation | Runner routing |
|------|--------|---------------------|----------------|
| `read_file` | `common/tools.py` | `read_file_from_disk()` → `/tmp/blitzy/{repo}/{branch}/{path}` | `routed_read_file_from_disk()` via `runner_session` from config |
| `bash` | Anthropic built-in | Local subprocess | `runner_session.run_bash()` — branched in each consumer's helper.py |

All other tools (Neo4j graph queries, GCS storage, in-memory lookups, web search) do NOT
touch the local filesystem and need no routing.

### How `read_file` routing works

**`archie-shared/common/tools.py`**: The `read_file` tool reads `runner_session` from
`config["configurable"].get("runner_session")` and calls `routed_read_file_from_disk()`
which routes to the runner when session is present, or falls back to local `read_file_from_disk()`.

**Consumer side**: Each consumer passes `"runner_session": self.runner_session` in the
`tools_config` dict that feeds into `process_tool_call()`. When `runner_session` is `None`
(USE_RUNNER=false), behavior is identical to pre-migration.

### How bash routing works

Bash is an Anthropic built-in tool — it doesn't go through `process_tool_call()`. Each
consumer intercepts it in the tool loop and branches:

```python
if tool_name == ANTHROPIC_BASH_TOOL_NAME:
    if restart:
        if self.runner_session is not None:
            self.runner_session.restart_session()
        else:
            self.bash_session, _ = await restart_bash_session(...)
    else:
        if self.runner_session is not None:
            tool_result = self.runner_session.run_bash(command)
        else:
            tool_result = await handle_bash_tool_response(...)
```

### Wrapper functions (`archie-shared`)

| Wrapper | Location | Purpose |
|---------|----------|---------|
| `routed_restart_bash_session()` | `common/bash.py` | Routes to `runner_session.restart_session()` or local `restart_bash_session()`. Same async `(bash_session, is_error)` return signature — drop-in replacement. |
| `routed_read_file_from_disk()` | `blitzy_utils/runner_ops/client.py` | Routes to runner op or local `read_file_from_disk()`. Used by `read_file` tool in `common/tools.py`. |

### Metering / billing — runner routing (CRITICAL)

Metering stats feed into notification payloads that drive **customer billing**. When runner is
active, repo files live on the worker pod — any metering that reads the local filesystem will
silently return 0/0.

**code-downloader notification payload** (`main.py:336`):
- `lines_onboarded` — via `routed_count_lines_in_file()` → **already runner-aware**
- `files_onboarded`, `total_files` — from `file_paths` list → **works** (list from routed download)
- `file_extensions` — computed in same loop → **works**
- `ValidationService.validate_onboarding_quota()` — uses `FileSystemRepository` (local `glob` + `open`) → **BROKEN with runner** — returns 0/0, quota check silently passes

**reverse-code-generator notification payload** (3 sites in `helper.py`):
- `files_modified`, `additions`, `edits`, `removals` — via `routed_get_commit_diff_summary()` → **already runner-aware**

**Fix for `ValidationService`**:
`FileSystemRepository` in `archie-shared/metering/repositories.py` is hardcoded to local disk.
It implements `IFileSystemRepository` (DI interface). The fix:
1. Create `RunnerFileSystemRepository(IFileSystemRepository)` — implements `find_files()`,
   `count_lines()`, `is_file()` via runner ops on the worker
2. Inject it into `ValidationService` when `runner_session` is active
3. Add runner ops on worker: `find_files`, `count_lines`, `is_file` (or composite `get_directory_file_details`)

### Per-consumer tool routing status

| Consumer | `read_file` routing | `bash` routing | `restart_bash_session` routing |
|----------|-------------------|----------------|-------------------------------|
| `archie-job-code-downloader` | N/A (no LLM tools) | N/A | N/A |
| `archie-job-code-graph-generator` | N/A (calls `routed_read_file_from_disk` directly) | N/A (no bash tool) | N/A |
| `archie-job-reverse-document-generator` | `runner_session` in `tools_config` → `read_file` routes via shared layer | Branched in helper.py | `routed_restart_bash_session()` in setup() |
| `archie-job-reverse-thinking-generator` | `runner_session` in `tools_config` → `read_file` routes via shared layer | N/A (no bash tool) | N/A (no bash session) |
| `archie-job-reverse-file-mapper` | Intercepted in tool loop (`remote_read_file_from_disk`) | Branched in helper.py | Branched in setup() |
| `archie-job-reverse-code-generator` | Custom `tools.py` uses `routed_read_file_from_disk` + `runner_session` in `tools_config` | Branched in helper.py | Branched in configure_local_vm() and elsewhere |

---

## Completed Migrations

### 1. `archie-job-code-downloader`

**Files**: `main.py`

**Pattern variant**: Flat function (`process_event()`), no helper class. Uses `use_runner` bool
and passes `session` as argument to sub-functions.

**Session lifecycle** (`main.py:89-93, 374-375`):
```python
use_runner = should_use_runner()
session = None
if use_runner:
    session = RunnerSession(BlitzyClient(), company_id, job_id)
    session.start()
# ... all processing ...
if session:
    session.stop()
```

**5 flag-branched call sites** using `remote_*` wrappers from `blitzy_utils.runner_ops.client`:

| # | Call site | Local function | Remote wrapper |
|---|-----------|---------------|----------------|
| 1 | `process_event()` | `download_repository_to_disk()` | `remote_get_all_files_from_cloned_repo(session)` |
| 2 | `process_event()` | `load_blitzyignore()` | `remote_read_blitzyignore(session)` + `_parse_blitzyignore_content()` |
| 3 | `find_reusable_branch()` | `find_closest_ancestor_branch()` | `remote_find_closest_ancestor_branch(session, ...)` |
| 4 | `download_and_upload()` | `get_changed_files_between_commits()` | `remote_get_changed_files_between_commits(session, ...)` |
| 5 | `download_and_upload()` | `count_lines_in_file()` | `remote_count_lines_in_file(session, path)` |

**Extras**:
- `_parse_blitzyignore_content()` — shared parser extracted so both local and remote paths
  produce the same `List[str]` output.
- `use_runner` bool and `session` are propagated as function arguments to `find_reusable_branch()`
  and `download_and_upload()`.

---

### 2. `archie-job-code-graph-generator`

**Files**: `lib/blitzy/code_graph/helper.py`

**Pattern variant**: `should_use_runner()` called in constructor. Session created in `setup()`
(LangGraph node), stopped in `teardown()` (LangGraph node). No external lifecycle management.

**Session lifecycle** (`helper.py:104-108, 243-249, 1437-1439`):
```python
# __init__
self.use_runner = should_use_runner()
self.runner_session = None

# setup() — LangGraph node
if self.use_runner:
    client = BlitzyClient()
    self.runner_session = RunnerSession(client, self.company_id, self.job_metadata.get("job_id", ""))
    self.runner_session.start()
else:
    download_repository_to_disk(...)

# teardown() — LangGraph node
if self.runner_session is not None:
    self.runner_session.stop()
    self.runner_session = None
```

**1 flag-branched call site** using `remote_read_file_from_disk` wrapper:

| # | Call site | Local function | Remote wrapper |
|---|-----------|---------------|----------------|
| 1 | `prepare_file()` | `read_file_from_disk(file_path, ...)` | `remote_read_file_from_disk(self.runner_session, file_path)` |

**Note**: This consumer only reads files — no bash tool calls. File listing comes from GCS
(`storage_service.download_repo_structure_batch()`), not the runner.

---

### 3. `archie-job-reverse-document-generator`

**Files**: `main.py`, `lib/reverse_document/helper.py`

**Pattern variant**: `main.py` owns the lifecycle with `try/finally`, passes `runner_session`
to helper constructor. Helper uses `self.runner_session is not None` as the flag (no separate
bool). Direct `run_bash()` / `restart_session()` calls — no `remote_*` wrappers needed.

**Session lifecycle** (`main.py:223-227, 229, 348-350`):
```python
runner_session = None
if should_use_runner():
    client = BlitzyClient()
    runner_session = RunnerSession(client, company_id, job_id)
    runner_session.start()

try:
    rd_helper = ReverseDocumentHelper(..., runner_session=runner_session)
    # ... graph execution ...
finally:
    if runner_session:
        runner_session.stop()
```

**5 flag-branched call sites**:

| # | Method | What | Runner replacement |
|---|--------|------|--------------------|
| 1 | `setup()` | `download_repository_to_disk(...)` | Skip |
| 2 | `setup()` | `await restart_bash_session(...)` | Skip |
| 3 | `gather_context()` | `await restart_bash_session(...)` | `self.runner_session.restart_session()` |
| 4 | `gather_context()` | `await handle_bash_tool_response(...)` | `self.runner_session.run_bash(command)` |
| 5 | `process_section()` | `await restart_bash_session(...)` | `self.runner_session.restart_session()` |
| 6 | `process_section()` | `await handle_bash_tool_response(...)` | `self.runner_session.run_bash(command)` |

---

## Remaining Migrations

### 4. `archie-job-reverse-file-mapper`

**Files to modify**: `main.py`, `lib/reverse_mapper/helper.py`

**Pattern**: Same as reverse-document-generator — `main.py` lifecycle, helper constructor param.

**Note**: Has `async def init()` (line 172) called after construction for Figma MCP tools.
RunnerSession creation should go before helper creation, not inside `init()`.

**Call sites to flag-branch**:

| # | Method | Line | Current | Runner replacement |
|---|--------|------|---------|--------------------|
| 1 | `setup()` | 268 | `download_repository_to_disk(...)` | Skip |
| 2 | `setup()` | 279 | `await restart_bash_session(...)` | Skip |
| 3 | `process_folder()` | 447 | `await restart_bash_session(...)` | `self.runner_session.restart_session()` |
| 4 | `process_folder()` | 454 | `await handle_bash_tool_response(...)` | `self.runner_session.run_bash(command)` |

**main.py changes**: Create `RunnerSession` before `ReverseMapperHelper(...)` (line 152),
pass as `runner_session=runner_session`, add `try/finally` with `runner_session.stop()`.

---

### 5. `archie-job-reverse-code-generator`

**Files to modify**: `main.py`, `lib/blitzy/helper.py`

**Pattern**: Same as reverse-document-generator, but more complex — has TWO repo downloads
(source + destination) and additional bash calls for git configuration.

**Call sites to flag-branch**:

| # | Method | Line | Current | Runner replacement |
|---|--------|------|---------|--------------------|
| 1 | `setup()` | 603 | `download_repository_to_disk(...)` (source repo) | Skip |
| 2 | `setup()` | 628 | `download_repository_to_disk(...)` (dest repo) | Skip |
| 3 | `configure_local_vm()` | 650 | `await restart_bash_session(...)` | Skip |
| 4 | `configure_local_vm()` | 660 | `await handle_bash_tool_response(...)` (git config) | `self.runner_session.run_bash(command)` |
| 5 | mid-processing | 1005 | `restart_bash_session(...)` | `self.runner_session.restart_session()` |
| 6 | LLM loop | 1165 | `await restart_bash_session(...)` | `self.runner_session.restart_session()` |
| 7 | LLM loop | 1175 | `await handle_bash_tool_response(...)` | `self.runner_session.run_bash(command)` |
| 8 | sync local state | 1356 | `restart_bash_session(...)` | `self.runner_session.restart_session()` |

**main.py changes**: Create `RunnerSession` before `ReverseCodeGeneratorHelper(...)` (line 232),
pass as `runner_session=runner_session`, add `try/finally` with `runner_session.stop()`.

**Open question**: `configure_local_vm()` runs `git config --global` via bash — this still
needs to execute on the runner (site #4). Sites #1-3 can be skipped entirely.
The `state["current_session_cmds"]` tracking (rolling window of last 12 commands) works
unchanged with the runner path.

---

### 6. `archie-job-reverse-thinking-generator`

**Files**: `main.py`, `lib/reverse_thinker/helper.py`

**Pattern**: Same as reverse-document-generator — `main.py` lifecycle, helper constructor param.
No bash tool in tool lists. Uses `read_file` tool via `process_tool_call()`.

**Already wired**:
- `RunnerSession` lifecycle in `main.py` (create, start, pass to helper, stop in finally)
- `runner_session` param in `ReverseThinkerHelper.__init__`
- `routed_download_repository_to_disk()` in `setup()`

**Tool routing** (added 2026-03-30):
- `"runner_session": self.runner_session` added to `tools_config` dict in `process_file()` (line 559)
- `read_file` tool routes via `archie-shared` shared layer — no consumer-side intercept needed

**Status**: Complete — runner_session lifecycle + tool routing done.

---

## Verification Checklist

For each migrated consumer:

- [ ] `USE_RUNNER=false` (default): behavior identical to pre-migration
- [ ] `USE_RUNNER=true`: `setup()` skips clone + local bash init
- [ ] `USE_RUNNER=true`: LLM bash tool calls route through `runner_session.run_bash()`
- [ ] `USE_RUNNER=true`: bash restart calls route through `runner_session.restart_session()`
- [ ] Runner session is stopped in all exit paths (try/finally or teardown node)
- [ ] No import errors — `py_compile` passes on all modified files
