# `blitzy_utils.runner_ops` — inventory & gaps for chat

## Already registered on the worker (no changes needed)

From `blitzy-utils-python/blitzy_utils/blitzy_utils/runner_ops/operations/`:

### `file_ops.py`
| op name | function | covers |
|---|---|---|
| `get_all_files_from_cloned_repo` | `list_files` | tree listing |
| `read_blitzyignore` | `read_blitzyignore` | .blitzyignore content |
| `read_file` | `read_file` | full file content |
| `write_file` | `write_file` | future write mode |
| `check_path_type` | `check_path_type` | exists / is-dir / is-file |
| `get_size` | `get_size` | file size |
| `write_binary_file` | `write_binary_file` | binary writes |
| `count_lines` | `count_lines` | per file |
| `count_lines_batch` | `count_lines_batch` | many files |
| `count_lines_from_file` | `count_lines_from_file` | path list from a file |
| `write_file_from_payload` | `write_file_from_payload` | atomic write via JSON payload |

### `git_ops.py`
| op name | function | covers |
|---|---|---|
| `find_ancestor` | `find_ancestor` | merge-base / closest ancestor |
| `changed_files` | `changed_files` | diff between commits |

### `scm_ops.py`
| op name | function | covers |
|---|---|---|
| `download_repository` | `download_repo` | **the clone — already wired into worker startup** |
| `setup_github_branch` | `setup_branch` | branch creation |
| `create_commit` | `create_commit` | commit |
| `head_commit` | `head_commit` | get HEAD sha |
| `create_prs` | `create_prs` | open PR(s) |
| `push_pull` | `push_pull` | sync |
| `commit_diff` | `commit_diff` | diff summary |

## Already wrapped client-side (`blitzy_utils.runner_ops.client.routed_*`)

`routed_download_repository_to_disk`, `routed_find_closest_ancestor_branch`,
`routed_get_changed_files_between_commits`, `routed_count_lines_in_file`,
`routed_count_lines_in_files`, `routed_get_commit_diff_summary`,
`routed_count_lines_from_file`, `routed_setup_github_branch`,
`routed_create_github_commit`, `routed_create_all_pull_requests`,
`routed_get_head_commit_hash`, `routed_push_pull_latest_from_repository`,
`routed_get_all_files_from_cloned_repo`, `routed_read_file`,
`routed_write_file`, `routed_check_path_type`, `routed_get_size`,
`routed_write_binary_file`, `routed_write_file_from_payload`,
`routed_remote_read_blitzyignore`. (Full signatures in
`blitzy-utils-python/blitzy_utils/blitzy_utils/runner_ops/client.py`.)

## Missing — need to add for chat phase 4

| Proposed op | Where | What it returns | Notes |
|---|---|---|---|
| `ls_dir` | `file_ops.py` | `[{"name":"x","type":"d|-"}]` | Mirrors `VirtualFilesystem.list_dir` |
| `glob_match` | `file_ops.py` | `["/path1", "/path2", ...]` | `fnmatch` against the cloned tree under a base |
| `find_files` | `file_ops.py` | `["/path1", ...]` | `find` semantics: name pattern, type filter, depth |
| `grep_files` | `text_ops.py` (new) | `[{"path":"/x","line_no":12,"line":"..."}, ...]` | One-shot recursive grep with `-rn`/`--include` |
| `cat_with_range` | `file_ops.py` | `"<text>"` | Saves a round-trip when only N lines are needed |
| `git_status` | `git_ops.py` | `{"clean": bool, "modified": [...]}` | Defensive read for future write mode |

Each is 30–80 lines; all delegate to existing helpers in
`blitzy_utils.git_helpers` or shell out via the worker's `BashSession`.

## Bash escape hatch (already exists)

`RunnerSession.run_bash(command)` (see `session.py:168`) lets the chat
service execute arbitrary shell on the worker without registering an op for
every command. Phase 4 uses this for `shell` tool: parse the agent's command,
run it via `run_bash`, return formatted stdout/stderr. Keep the per-command
ops above for hot paths (read, ls, grep) where a typed op is faster and more
reliable.

## Worker capability negotiation (recommended in phase 4)

Add `runner_op("describe_capabilities")` returning `{ "ops": [...names...],
"protocol_version": "1.0" }`. `RunnerSession.start()` calls this once and
records the set; `routed_*` wrappers can then refuse to call ops the worker
doesn't have (and either fall back to local exec or surface a clean error).
This prevents silent failures on version skew between mediator-deployed
worker image and the chat service's expected ops.
