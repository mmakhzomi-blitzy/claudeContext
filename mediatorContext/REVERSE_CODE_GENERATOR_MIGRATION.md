# Reverse-Code-Generator Migration to Runner Operations

## Context

The reverse-code-generator is a LangGraph-based agentic job that generates code in a destination repo from a source repo. It currently uses local `BashSession` (asyncio subprocess) for bash commands and direct function calls for SCM operations and file I/O. All these operations need to run on the remote K8s worker via the runner_ops framework.

Unlike code-downloader (sequential steps, single repo), reverse-code-generator has:
- **Agentic loop**: LLM drives tool calls (bash, text_editor) in a loop
- **Two repos**: source (read-only) + destination (write target)
- **Long-running bash session**: multiple commands throughout the job
- **SCM API calls**: GitHub branch setup, commits, PRs — all need credentials available on the worker

The worker auto-clone (`startup.py`) only clones ONE repo from EVENT_DATA. Since reverse-code-generator needs TWO repos and other consumers should also explicitly control cloning, we'll make `download_repository_to_disk` a proper runner_op and remove the auto-clone.

**Scope**: Migrate bash commands, file I/O, and SCM operations to the runner. Chrome DevTools MCP and Figma MCP are out of scope — they remain on the consumer pod for now (see "Out of Scope" section).

---

## Implementation Order

1. New runner_ops + routed functions (blitzy-utils-python)
2. Update existing runner_ops for multi-repo support (blitzy-utils-python)
3. Worker auto-clone removal (archie-client-worker)
4. Reverse-code-generator migration (archie-job-reverse-code-generator)
5. Other consumer verification (code-downloader, code-graph-generator)

---

## Phase 1: New Runner Operations (blitzy-utils-python)

### 1a: New SCM operations file

**New file**: `blitzy_utils/runner_ops/operations/scm_ops.py`

Register these `@runner_op` functions. Each wraps the corresponding `scm.py` function, accepting ALL params explicitly (not from context) so callers can specify any repo:

| Operation Name | Wraps | Key Params | Returns |
|---|---|---|---|
| `download_repository` | `scm.download_repository_to_disk()` | repo_name, branch_name, user_id, server, commit_hash, git_project_repo_id, repo_id, overwrite_existing_folder, clean | `bool` (clone only, no file list) |
| `setup_github_branch` | `scm.setup_github_branch()` | repo_name, user_id, server, repo_id, branch_name, base_branch, create_new_branch, delete_existing_branch, is_new_repo, git_project_repo_id | `str` |
| `create_github_commit` | `scm.create_github_commit()` | repo_name, repo_id, branch_name, base_branch, file_path, head_commit_hash, user_id, content, create_new_branch, delete_file, is_new_repo, server, commit_message, git_project_repo_id | `dict \| None` (serialized commit) |
| `get_head_commit_hash` | `scm.get_head_commit_hash()` | repo_name, user_id, server, branch_name, repo_id, git_project_repo_id | `str` |
| `create_all_pull_requests` | `scm.create_all_pull_requests()` | repo_name, repo_id, head_branch, user_id, server, base_branch, pr_title, pr_body, is_new_repo, git_project_repo_id | `list[dict]` (serialized PR objects) |
| `push_pull_latest_from_repository` | `scm.push_pull_latest_from_repository()` | repo_name, branch_name, user_id, server, repo_id, git_project_repo_id | `bool` |
| `get_commit_diff_summary` | `metering.get_commit_diff_summary()` | disk_path, base_commit, target_commit, exclude_blank_lines | `dict` (CommitDiffSummary is already a TypedDict) |

**Serialization for complex return types**:
- `create_github_commit`: Return `{"sha": commit.sha}` or `None`
- `create_all_pull_requests`: Return `[{"html_url": pr.html_url, "number": pr.number, "title": pr.title, ...}]`
- `download_repository`: Clone only — returns `True` on success. File listing is a separate operation via the existing `get_all_files_from_cloned_repo` runner_op.

### 1b: New `write_file_to_disk` operation

**File**: `blitzy_utils/runner_ops/operations/file_ops.py` (existing)

```python
@runner_op("write_file_to_disk")
def write_file(context: RunnerContext, *, file_path: str, file_text: str,
               repo_name: str = None, branch_name: str = None, **kwargs) -> bool:
    disk.write_file_to_disk(
        file_path=file_path, file_text=file_text,
        repo_name=repo_name or context.repo_name,
        branch_name=branch_name or context.branch_name,
    )
    return True
```

### 1c: Fix `read_file_from_disk` remote path to forward repo params

**File**: `blitzy_utils/runner_ops/operations/file_ops.py`

The existing `read_file_from_disk` runner_op ignores caller-provided `repo_name`/`branch_name` and always reads from the EVENT_DATA repo. The caller already passes these params (they're required in `routed_read_file_from_disk`), but the remote path drops them. Fix all three layers:

1. **Runner_op** (`file_ops.py`): Accept optional `repo_name`/`branch_name`, default to context:
   ```python
   @runner_op("read_file_from_disk")
   def read_file(context, *, file_path, repo_name=None, branch_name=None, **kwargs):
       read_file_from_disk(
           file_path=file_path,
           repo_name=repo_name or context.repo_name,
           branch_name=branch_name or context.branch_name,
       )
   ```

2. **Remote helper** (`client.py`): Forward params:
   ```python
   def remote_read_file_from_disk(session, file_path, repo_name=None, branch_name=None):
       result = session.run("read_file_from_disk", file_path=file_path,
                            repo_name=repo_name, branch_name=branch_name)
   ```

3. **Routed function** (`client.py`): Pass to remote helper:
   ```python
   if use_runner:
       return remote_read_file_from_disk(session, file_path, repo_name, branch_name)
   ```

No consumer changes needed — they already pass `repo_name`/`branch_name`.

### 1d: Update operations `__init__.py`

**File**: `blitzy_utils/runner_ops/operations/__init__.py`

Add: `from . import scm_ops  # noqa: F401`

### 1e: New routed functions

**File**: `blitzy_utils/runner_ops/client.py`

Add routed wrappers following the existing pattern (check `should_use_runner()`, route to `session.run()` or local function):

- `routed_setup_github_branch(...)` -> remote: `session.run("setup_github_branch", ...)`
- `routed_create_github_commit(...)` -> remote: `session.run("create_github_commit", ...)`
- `routed_get_head_commit_hash(...)` -> remote: `session.run("get_head_commit_hash", ...)`
- `routed_create_all_pull_requests(...)` -> remote: `session.run("create_all_pull_requests", ...)`
- `routed_push_pull_latest_from_repository(...)` -> remote: `session.run("push_pull_latest_from_repository", ...)`
- `routed_write_file_to_disk(...)` -> remote: `session.run("write_file_to_disk", ...)`
- `routed_get_commit_diff_summary(...)` -> remote: `session.run("get_commit_diff_summary", ...)`

**Update existing** `routed_download_repository_to_disk()`:
- Change remote path to two separate calls:
  1. `session.run("download_repository", repo_name=..., branch_name=..., ...)` — clone the repo
  2. `remote_get_all_files_from_cloned_repo(session)` — list files from the cloned repo
- This keeps clone and file-listing as separate operations. Consumers that only need the clone (e.g., reverse-code-generator for dest repo) can call `session.run("download_repository", ...)` directly without listing files.

---

## Phase 2: Worker Auto-Clone Removal (archie-client-worker)

**File**: `archie-client-worker/main.py`
- Remove `.with_repo_clone()` from the builder chain:
  ```python
  # Before: WorkerStartupBuilder(settings).with_repo_clone().build()
  # After:  WorkerStartupBuilder(settings).build()
  ```

**File**: `archie-client-worker/src/blitzy_worker/startup.py`
- Keep `CloneRepositoryStep` class, just remove it from the builder chain

All consumers now explicitly clone via `routed_download_repository_to_disk()` which calls the `download_repository` runner_op on the worker.

---

## Phase 3: Reverse-Code-Generator Migration

### 3a: main.py

**File**: `archie-job-reverse-code-generator/main.py`

- Pass `runner_session` to `ReverseCodeGeneratorHelper` constructor
- Already creates `RunnerSession` when `should_use_runner()` is True (existing code)

### 3b: helper.py — Constructor

**File**: `archie-job-reverse-code-generator/lib/blitzy/helper.py`

- Add `runner_session: RunnerSession | None = None` param to `__init__`
- Store as `self.runner_session`
- Update imports: add routed functions from `blitzy_utils.runner_ops.client`
- Add code comment near MCP initialization (around `self.mcp_manager` / Chrome / Figma setup):
  ```python
  # NOTE: Chrome DevTools MCP and Figma MCP are NOT supported in remote runner mode.
  # Both run as local stdio subprocesses and require filesystem access to the dest repo
  # working directory. When runner_session is active, the dest repo lives on the worker
  # pod, but MCP servers run on this consumer pod — they cannot reach the worker's
  # filesystem or localhost. MCP co-location with the worker is a separate migration task.
  ```

### 3c: helper.py — setup() method (lines 558-681)

Replace each operation with its routed equivalent:

| Current | Replacement |
|---|---|
| `setup_github_branch(...)` | `routed_setup_github_branch(..., runner_session=self.runner_session)` |
| `download_repository_to_disk(source_params)` | `routed_download_repository_to_disk(source_params, runner_session=self.runner_session)` |
| `download_repository_to_disk(dest_params)` | `routed_download_repository_to_disk(dest_params, runner_session=self.runner_session)` |
| `get_head_commit_hash(...)` | `routed_get_head_commit_hash(..., runner_session=self.runner_session)` |
| `os.makedirs(screenshots_path)` | `self.runner_session.run_bash(f"mkdir -p {path}")` if remote, else `os.makedirs()` |

### 3d: helper.py — configure_local_vm() (line 676)

| Current | Replacement (remote mode) |
|---|---|
| `restart_bash_session(bash_session, repo_name, branch_name)` | `self.runner_session.restart_session()` |
| Bash git config commands via BashSession | `self.runner_session.run_bash("git config ...")` |

In local mode, keep existing BashSession flow. Pattern:
```python
if self.runner_session:
    self.runner_session.restart_session()
    self.runner_session.run_bash("git config --global user.email 'blitzy@blitzy.com'")
    self.runner_session.run_bash("git config --global user.name 'Blitzy'")
else:
    self.bash_session, _ = await restart_bash_session(...)
    # existing bash config
```

### 3e: helper.py — sync_local_git_state() (line 1387)

| Current | Replacement (remote mode) |
|---|---|
| `push_pull_latest_from_repository(...)` | `routed_push_pull_latest_from_repository(..., runner_session=self.runner_session)` |
| `handle_bash_tool_response(command="git status --porcelain")` | `self.runner_session.run_bash("git status --porcelain")` |

### 3f: helper.py — Agentic loop bash handling (lines 1190-1217)

The agentic loop dispatches LLM tool calls. For bash tool:

```python
# Current:
if restart:
    self.bash_session, _ = await restart_bash_session(...)
else:
    result = await handle_bash_tool_response(bash_session=self.bash_session, command=command)

# New:
if self.runner_session:
    if restart:
        self.runner_session.restart_session()
    else:
        result = self.runner_session.run_bash(command)
else:
    # existing local path unchanged
```

Note: `runner_session.run_bash()` is sync (polls for completion). Calling from async context is fine since the agentic loop is sequential — no concurrent async work.

### 3g: helper.py — Text editor tool handling (lines 1069-1190)

For file read/write in text editor tools, pass `runner_session` to tools.py functions:

| Current | Replacement |
|---|---|
| `handle_text_editor_view_command(...)` | `handle_text_editor_view_command(..., runner_session=self.runner_session)` |
| `handle_text_editor_str_replace_command(...)` | `handle_text_editor_str_replace_command(..., runner_session=self.runner_session)` |
| `handle_text_editor_insert_command(...)` | `handle_text_editor_insert_command(..., runner_session=self.runner_session)` |
| Direct `write_file_to_disk(...)` calls | `routed_write_file_to_disk(..., runner_session=self.runner_session)` |

### 3h: helper.py — Post-processing & teardown

| Current | Replacement |
|---|---|
| `get_commit_diff_summary(disk_path, base_commit)` | `routed_get_commit_diff_summary(disk_path, base_commit, runner_session=self.runner_session)` |
| `create_github_commit(...)` x2 (guide + tech spec) | `routed_create_github_commit(..., runner_session=self.runner_session)` |
| `create_all_pull_requests(...)` | `routed_create_all_pull_requests(..., runner_session=self.runner_session)` |

### 3i: tools.py

**File**: `archie-job-reverse-code-generator/lib/blitzy/tools.py`

Add `runner_session=None` param to these functions:
- `handle_text_editor_view_command()` -> use `routed_read_file_from_disk(..., runner_session=runner_session)`
- `handle_text_editor_str_replace_command()` -> use `routed_read_file_from_disk` + `routed_write_file_to_disk`
- `handle_text_editor_insert_command()` -> use `routed_write_file_to_disk`
- `try_get_source_or_dest_file()` -> use `routed_read_file_from_disk`

---

## Phase 4: Other Consumer Verification

### 4a: Code-downloader

**File**: `archie-job-code-downloader/main.py`

Already uses `routed_download_repository_to_disk()`. After Phase 1 update (remote path now calls `download_repository` runner_op instead of `get_all_files_from_cloned_repo`), it will explicitly clone on the worker. No consumer code changes needed — just verify it works.

### 4b: Code-graph-generator

**File**: `archie-job-code-graph-generator/main.py`

Needs investigation: does it read files from disk on the worker, or from GCS? If from disk, it needs an explicit `routed_download_repository_to_disk` call added. If from GCS only, no changes needed.

### 4c: Other consumers (reverse-document-generator, reverse-file-mapper)

Same pattern — if they use runner and read from disk, they need explicit download calls. Verify during rollout.

---

## Key Files Modified

| Repo | File | Change |
|---|---|---|
| blitzy-utils-python | `runner_ops/operations/scm_ops.py` | **NEW** — 7 SCM runner_ops |
| blitzy-utils-python | `runner_ops/operations/file_ops.py` | Add `write_file_to_disk` op, fix `read_file_from_disk` to forward `repo_name`/`branch_name` |
| blitzy-utils-python | `runner_ops/operations/__init__.py` | Import `scm_ops` |
| blitzy-utils-python | `runner_ops/client.py` | 7 new routed functions, update `routed_download_repository_to_disk` remote path (separate clone + list-files), fix `remote_read_file_from_disk` + `routed_read_file_from_disk` to forward repo params |
| archie-client-worker | `main.py` | Remove `.with_repo_clone()` |
| archie-client-worker | `src/blitzy_worker/startup.py` | Optional: delete `CloneRepositoryStep` |
| reverse-code-generator | `main.py` | Pass `runner_session` to helper |
| reverse-code-generator | `lib/blitzy/helper.py` | Accept `runner_session`, route all SCM/bash/file ops |
| reverse-code-generator | `lib/blitzy/tools.py` | Accept `runner_session`, route file I/O |

---

## Verification

1. **blitzy-utils-python**: Run existing tests, verify new runner_ops register correctly via `python -m blitzy_utils.runner_ops` (should list all ops)
2. **Worker**: Deploy without auto-clone, verify worker starts and enters BLPOP loop
3. **Code-downloader regression**: Run a code-download job — verify repo is cloned via `download_repository` runner_op and files are returned
4. **Reverse-code-generator**: Run job with `should_use_runner()=True` — verify:
   - Both repos cloned on worker
   - Bash commands execute remotely
   - File reads/writes go through runner
   - SCM operations (branch setup, commits, PRs) succeed
   - Agentic loop functions correctly end-to-end
5. **Local fallback**: Run with `should_use_runner()=False` — verify no regressions in local mode

---

## Out of Scope / To-Do

### MCP Co-Location with Worker

Chrome DevTools MCP and Figma MCP currently run as local subprocesses (`stdio` transport) on the consumer pod. Both need access to the dest repo filesystem:

- **Chrome DevTools MCP**: Chrome browser loads the dev server (started via bash in the agentic loop) at `localhost`. If bash runs on the worker, Chrome on the consumer can't reach the worker's `localhost`.
- **Figma MCP**: Downloads assets (SVGs/PNGs) from the Figma API and saves them into the dest repo working directory. If the dest repo is on the worker, Figma MCP needs to write there too.

**Current plan**: Leave MCP servers on the consumer pod unchanged. Only migrate bash, file I/O, and SCM operations to the runner. MCP-related features (screenshots, Figma asset download) will continue to work against the consumer's local filesystem.

**Future work needed**: Either move MCP servers to the worker pod (and route tool calls through a new mechanism), or expose the worker's dev server via K8s service/port-forward so the consumer's Chrome can reach it. Figma assets could be downloaded via a runner_op. This needs a separate design pass.
