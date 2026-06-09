# Runner Operations Framework

## Overview

The runner operations framework lets GCP consumers execute filesystem and git operations on a remote runner (K8s worker pod) through the existing mediator infrastructure. Operations are Python functions decorated with `@runner_op` that wrap existing `blitzy_utils` code. The runner executes them via `python -m blitzy_utils.runner_ops <op> [kwargs]` through its BashSession — zero runner or mediator changes needed.

**Package location**: `blitzy-utils-python/blitzy_utils/blitzy_utils/runner_ops/`

---

## Package Structure

```
runner_ops/
├── __init__.py           — re-exports RunnerSession, RunnerOperationError
├── session.py            — RunnerSession class + RunnerOperationError
├── client.py             — should_use_runner() + remote_ wrappers + routed_ wrappers
├── __main__.py           — CLI entry point (runs on worker)
├── context.py            — RunnerContext dataclass (parsed from EVENT_DATA env var)
├── registry.py           — @runner_op decorator + OPERATIONS dict
└── operations/
    ├── __init__.py        — auto-imports file_ops, git_ops, scm_ops for registration
    ├── file_ops.py        — file system ops (read/write text & binary, list, count lines, file type)
    ├── git_ops.py         — git/branch utilities (ancestor branch, changed files)
    └── scm_ops.py         — SCM clone/push/PR ops (download repo, branch setup, commit, PR creation, diff)
```

---

## How It Works

```
Consumer (GCP)                              Worker Pod (K8s)
──────────────                              ────────────────
session.run("count_lines_in_file", file_path="src/a.py")
  │
  ├─ _build_command() →
  │   "python -m blitzy_utils.runner_ops count_lines_in_file '{"file_path":"src/a.py"}'"
  │
  ├─ create_job_command() → mediator → Redis LPUSH → worker BLPOP
  │                                                      │
  │                                        __main__.py executes:
  │                                          1. Parse args
  │                                          2. Load RunnerContext from EVENT_DATA
  │                                          3. Look up "count_lines_in_file" in OPERATIONS
  │                                          4. Call count_lines_in_file(context, file_path="src/a.py")
  │                                          5. stdout: {"status":"ok","result":42}
  │                                                      │
  ├─ poll get_command_status() ← mediator captures stdout┘
  │
  ├─ _parse_result() → json.loads(stdout) → extract "result"
  └─ returns 42
```

---

## Operation Registration

Operations register themselves at import time via the `@runner_op` decorator. When Python imports a module, it executes all top-level code including decorator calls.

```python
@runner_op("count_lines_in_file")          # runs at import time, NOT at call time
def count_lines_in_file(context, ...):
    ...
```

This is equivalent to:

```python
def count_lines_in_file(context, ...):
    ...
count_lines_in_file = runner_op("count_lines_in_file")(count_lines_in_file)
#                     ↑ calls runner_op()              ↑ calls returned decorator()
#                       which returns                    which does:
#                       decorator()                      OPERATIONS["count_lines_in_file"] = count_lines_in_file
```

The import chain: `__main__.py` → `operations/__init__.py` → `file_ops.py` + `git_ops.py` → all `@runner_op` decorators fire → `OPERATIONS` dict is populated.

---

## Registered Operations

### file_ops

| Operation | Wraps | Returns |
|-----------|-------|---------|
| `get_all_files_from_cloned_repo` | `get_all_files_from_cloned_repo()` | `[{"path": "...", "text": "..."}]` |
| `read_blitzyignore` | `read_file_from_disk(".blitzyignore")` | `str` |
| `count_lines_in_file` | `count_lines_in_file()` (single file) | `int` |
| `count_lines_in_files` | `count_lines_in_files()` (batch) | `dict[path, int]` |
| `read_file_from_disk` | `read_file_from_disk(path)` | `str` |
| `write_file_to_disk` | `write_file_to_disk(path, content)` | `bool` |
| `write_binary_file` | base64-decode and write bytes to absolute path (used for MCP asset push: Chrome screenshots, Figma downloads) | `bool` |
| `check_path_type` | `check_path_type(path)` — file/dir/missing | `str` |
| `get_file_size` | `get_file_size(path)` | `int` |

### git_ops

| Operation | Wraps | Returns |
|-----------|-------|---------|
| `find_closest_ancestor_branch` | `find_closest_ancestor_branch()` | `{"branch_id": ..., "head_commit_hash": ...}` or `None` |
| `get_changed_files_between_commits` | `get_changed_files_between_commits()` | `["file1.py", "file2.py", ...]` |

### scm_ops

| Operation | Wraps | Returns |
|-----------|-------|---------|
| `download_repository` | clone repo into runner's working directory | repo metadata dict |
| `setup_github_branch` | checkout or create branch | branch info dict |
| `create_github_commit` | stage + commit files | commit SHA |
| `get_head_commit_hash` | read current HEAD | `str` |
| `create_all_pull_requests` | open PRs across configured branches | list of PR URLs |
| `push_pull_latest_from_repository` | sync local with remote | bool |
| `get_commit_diff_summary` | summarize a commit's file changes | dict |

---

## RunnerSession

Long-lived session — one per job. Manages runner lifecycle and provides `run()` / `run_parallel()` for executing operations.

### Lifecycle

```python
from blitzy_utils.blitzy_client import BlitzyClient
from blitzy_utils.runner_ops import RunnerSession

client = BlitzyClient()

with RunnerSession(client, company_id, job_id) as session:
    result = session.run("get_all_files_from_cloned_repo")
# runner cleaned up automatically if this session created it
```

`start()` checks runner status:
- **READY / RUNNING** → reuse existing runner
- **DEPLOYING** → wait for ready
- **Not found** → create + wait

`stop()` always deletes the runner. Whoever creates the session is responsible for cleanup.

### Single Operation

```python
files = session.run("get_all_files_from_cloned_repo")
count = session.run("count_lines_in_file", file_path="src/main.py")
```

Submits command, polls until complete, parses JSON result.

### Parallel Operations

```python
files, ignore, ancestor = session.run_parallel([
    ("get_all_files_from_cloned_repo", {}),
    ("read_blitzyignore", {}),
    ("find_closest_ancestor_branch", {
        "completed_branches": [...],
        "new_head_commit_hash": "abc123",
    }),
])
```

Submits all commands upfront, then polls until all reach terminal status. Results returned in input order.

Tracks by **list index + execution_id** (not operation name), so duplicate operation names work:

```python
line_counts = session.run_parallel([
    ("count_lines_in_file", {"file_path": "src/a.py"}),   # index 0
    ("count_lines_in_file", {"file_path": "src/b.py"}),   # index 1
    ("count_lines_in_file", {"file_path": "src/c.py"}),   # index 2
])
# → [42, 17, 88]
```

### Raw Bash

```python
output = session.run_bash("ls -la /tmp/blitzy")
```

### Restart Session

```python
session.restart_session()
```

---

## Thread Safety

`run()` and `run_parallel()` use only local variables — no shared mutable state is accessed during execution. Multiple threads can call these methods concurrently on the same session.

Shared state:
- `_client`, `_company_id`, `_job_id`, `_poll_interval`, `_max_poll_interval` — immutable after `__init__`
- No mutable shared state — `stop()` always cleans up

---

## Adding a New Operation

One decorated function. No runner, mediator, or infrastructure changes.

```python
# operations/file_ops.py (or a new module)

@runner_op("read_file")
def read_file(context: RunnerContext, *, file_path: str, **kwargs: Any) -> str:
    return read_file_from_disk(file_path, context.repo_name, context.branch_name)
```

If adding a new module, import it in `operations/__init__.py`:

```python
from . import file_ops   # noqa: F401
from . import git_ops    # noqa: F401
from . import new_ops    # noqa: F401  ← add this
```

Consumer usage:

```python
content = session.run("read_file", file_path="src/main.py")
```

---

## Routed Wrappers

The `routed_` functions in `client.py` eliminate flag-branching in consumers. Each wrapper has the **same signature as the original function** plus an optional `runner_session` parameter. When `runner_session` is provided, it routes to the remote path. When `None`, it calls the original local function.

### Available Wrappers

| Routed Function | Notes |
|---|---|
| `routed_download_repository_to_disk` | clones repo on runner if session present, locally otherwise |
| `routed_find_closest_ancestor_branch` | git ancestor lookup |
| `routed_get_changed_files_between_commits` | git diff |
| `routed_count_lines_in_file` | single-file line count |
| `routed_count_lines_in_files` | batched line count (added for code-downloader perf) |
| `routed_read_file_from_disk` | read text |
| `routed_write_file_to_disk` | write text |
| `routed_push_binary_file` | push base64-encoded bytes to runner (MCP screenshot + Figma asset path) |
| `routed_check_path_type` | path is file / dir / missing |
| `routed_get_file_size` | bytes |
| `routed_setup_github_branch` | branch checkout/create |
| `routed_create_github_commit` | stage + commit |
| `routed_get_head_commit_hash` | HEAD SHA |
| `routed_create_all_pull_requests` | open PRs |
| `routed_push_pull_latest_from_repository` | sync with remote |
| `routed_get_commit_diff_summary` | commit diff summary |

All wrappers share the convention: signature matches the original local function plus an optional `runner_session` kwarg. When `runner_session` is provided, the call routes to the remote runner pod via `session.run(...)`; otherwise the local in-process function runs.

### Usage

```python
from blitzy_utils.runner_ops.client import (
    should_use_runner,
    routed_download_repository_to_disk,
    routed_count_lines_in_file,
)
from blitzy_utils.runner_ops.session import RunnerSession
from blitzy_utils.blitzy_client import BlitzyClient

# Create session only when USE_RUNNER=true; otherwise session stays None
session = None
if should_use_runner():
    session = RunnerSession(BlitzyClient(), company_id, job_id)

try:
    if session:
        session.start()

    # Single call — routes automatically based on session
    git_files = routed_download_repository_to_disk(
        repo_name=repo_name,
        branch_name=branch_name,
        user_id=user_id,
        server=server,
        commit_hash=commit_hash,
        git_project_repo_id=git_project_repo_id,
        runner_session=session,  # None → local, session → remote
    )

    count = routed_count_lines_in_file(
        file_path="src/main.py",
        repo_name=repo_name,
        branch_name=branch_name,
        runner_session=session,
    )
finally:
    if session:
        session.stop()
```

This replaces the previous flag-branching pattern where every call site needed:
```python
if use_runner:
    result = remote_x(session, ...)
else:
    result = local_x(...)
```

---

## Stdout Handling

`__main__.py` redirects all logging to **stderr** before running any operation, keeping stdout clean for JSON results. Without this, structured log lines from underlying functions (e.g., `get_all_files_from_cloned_repo`) would pollute stdout and break JSON parsing.

`session.py._clean_stdout()` strips any `EXIT_CODE=` lines from stdout before parsing. These are appended by the BashSession sentinel wrapper (`echo "EXIT_CODE=$EXIT_CODE {sentinel}"`) — the sentinel itself gets stripped by BashSession, but the `EXIT_CODE=` prefix remains.

---

## Error Handling

Operations return JSON via stdout:
- Success: `{"status": "ok", "result": <any>}`
- Failure: `{"status": "error", "error": "<message>"}`

`RunnerSession._parse_result()` raises `RunnerOperationError` on:
- Command status is not `COMPLETED` (FAILED, TIMEOUT, CANCELLED)
- stdout is not valid JSON
- JSON has `"status": "error"`

```python
from blitzy_utils.runner_ops import RunnerOperationError

try:
    result = session.run("find_closest_ancestor_branch", ...)
except RunnerOperationError as e:
    print(e.operation)     # "find_closest_ancestor_branch"
    print(e.execution_id)  # "exec-abc-123"
    print(str(e))          # "find_closest_ancestor_branch: <error message>"
```

---

## RunnerContext

`context.py` defines the `RunnerContext` dataclass, parsed from the `EVENT_DATA` env var on the worker. Field names match the main server's `job_metadata` payload.

**Required fields**: `repo_name`, `branch_name`, `user_id`, `head_commit_hash`, `git_project_repo_id`

**Optional fields**: `branch_id`, `repo_id`, `server`, `environments_ids`, `blitzy_job_id`

---

## Open Items

- **BlitzyClient thread safety** — URL cache needs lock or `lru_cache`
- **stdout size limits** — worker has 1MB/1000-line cap, large `get_all_files_from_cloned_repo` results may exceed this
- **Operation versioning** — consumer vs worker `blitzy_utils` version mismatch
- **Timeout configuration** — per-operation timeouts vs default 300s
