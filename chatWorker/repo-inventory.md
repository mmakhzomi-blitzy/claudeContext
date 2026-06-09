# Repo inventory — chat-worker plan

Quick map of which file in which repo. All paths absolute.

## archie-service-chat (qa branch)

| File | Role |
|---|---|
| `/Users/m.makhzomi/Projects/blitzy-platform/archie-service-chat/main.py:51` | Quart app instantiation |
| `…/main.py:175` | `verify_token_required` (RS256 / Firebase) |
| `…/main.py:299 build_context_from_req` | Pulls thread params from query string |
| `…/main.py:328 get_proj_uid_from_req` | Project owner UUID |
| `…/main.py:340 check_session_lock` | Per-thread session lock in Redis (TTL 600s) |
| `…/main.py:529 /resume` | SSE resume |
| `…/main.py:665 /start` | SSE start |
| `…/main.py:777 /stop` | Sets break_agent flag |
| `…/main.py:806 /close_session` | **Insertion point for `chat_worker.release()`** |
| `…/main.py:879 /clear` | Wipes thread state |
| `…/main.py:1052 /clear-cache` | Tool cache only |
| `…/main.py:1109 /health` | Liveness |
| `…/src/chat_server.py:226 bootstrap_workflow_context` | Builds `tools_config` — **insertion point for `runner_session`** |
| `…/src/chat_server.py:501 platform_tools` | Tool registration list |
| `…/src/chat_server.py:528 explorer_tools` | Worker agent tool subset |
| `…/src/chat_server.py:2309 stop_agent` | Stop logic — does **not** delete worker |
| `…/src/file_tools.py:25 download_file` tool | Calls `blitzy_utils.scm.download_single_file` |
| `…/src/vfs/filesystem.py:21 VirtualFilesystem` | Fake filesystem; `read_file` calls `download_single_file` |
| `…/src/vfs/shell.py:578 shell` tool | In-process pipeline executor |
| `…/src/vfs/commands.py` | All VFS commands (cmd_ls, cmd_grep, etc.) |
| `…/src/services/access_control.py` | Validates context |
| `…/src/services/status_queue.py` | SSE status events |
| `…/src/services/tool_cache.py` | Per-project tool cache |
| `…/src/models/chat_context.py:81 ChatContext` | Holds `to_lg_thread_id()` — the lease key |
| `…/src/consts.py` | `VFS_*`, `WORKER_AGENT_MODEL`, `TOOL_CACHE_TTL_SECONDS`, etc. |
| `…/env_config/env-{dev,qa}.yaml` | Per-env config |

## archie-client-mediator

| File | Role |
|---|---|
| `/Users/m.makhzomi/Projects/archie-client-mediator/src/api/routes/runners.py:74` | `POST /runners` (existing) |
| `…/src/api/routes/runners.py:143` | `DELETE /runners/<job_id>` |
| `…/src/api/routes/runners.py:197` | `GET /runners/<job_id>` (status) |
| `…/src/api/routes/commands.py` | Command submission |
| `…/src/api/models.py:115 RunnerConfig` | Resource & image config |
| `…/src/api/models.py:576 CreateRunnerRequest` | Existing request shape |
| `…/src/api/models.py:605 CreateRunnerResponse` | Existing response shape |
| `…/src/services/runner_service.py:31 RunnerService` | Orchestrates create/delete |
| `…/src/services/kubernetes_service.py:30 KubernetesService` | k8s client wrapper |
| `…/src/services/kubernetes_service.py:228 _build_deployment_spec` | Builds `V1Deployment` from `RunnerConfig` |
| `…/src/consts.py:33 WORKER_IMAGE` | env-injected default image |
| `…/src/consts.py:62 WORKER_SERVICE_ACCOUNT` | "client-mediator-service-account" |
| `…/main.py` | App startup — **insertion point for reaper task** |
| `…/swagger.yaml` | Source for `models.py` (datamodel-codegen) |

## archie-client-worker

| File | Role |
|---|---|
| `/Users/m.makhzomi/Projects/archie-client-worker/Dockerfile` | `python:3.12-slim` base; git, openssh-client, jq, Chrome |
| `…/main.py:1` | Entrypoint; signal handling |
| `…/src/blitzy_worker/startup.py:47 CloneRepositoryStep` | **Already clones from `EVENT_DATA`** |
| `…/src/blitzy_worker/startup.py:235 with_repo_clone` | Builder enabling clone step |
| `…/src/blitzy_worker/worker.py BlitzyWorker` | Polling loop |
| `…/src/blitzy_worker/processor.py` | Command processor |
| `…/src/blitzy_worker/bash_session.py` | Bash session adapter |
| `…/src/blitzy_worker/config.py Settings` | Env-driven config |

## archie-helm-chart

| File | Role |
|---|---|
| `/Users/m.makhzomi/Projects/archie-helm-chart/blitzy-client/chart/values.yaml:33 images` | Chart-wide image refs |
| `…/blitzy-client/chart/values-dev.yaml:13 clientWorker` | Stage worker image |
| `…/blitzy-client/chart/values-qa.yaml:16` | QA worker image |
| `…/blitzy-client/chart/values-prod.yaml` | Prod worker image |
| `…/blitzy-client/chart/templates/configmap.yaml:36 WORKER_IMAGE` | Injects image into mediator |
| `…/blitzy-client/chart/templates/mediator-sa.yaml` | Mediator's k8s SA + RBAC |
| `…/blitzy-client/chart/templates/client-mediator.yaml` | Mediator deployment |
| `…/helm-chart/apps/archie-service-chat/values.yaml` | Chat service deployment |

## blitzy_utils (sibling repo blitzy-utils-python)

| File | Role |
|---|---|
| `/Users/m.makhzomi/Projects/blitzy-utils-python/blitzy_utils/blitzy_utils/blitzy_client.py:232 BlitzyClient` | HTTP client to mediator (via relay) |
| `…/blitzy_utils/runner_ops/session.py:21 RunnerSession` | Lease abstraction — directly reusable |
| `…/blitzy_utils/runner_ops/client.py` | `routed_*` helpers (clone, file read/write, …) |
| `…/blitzy_utils/runner_ops/registry.py` | `@runner_op` decorator |
| `…/blitzy_utils/runner_ops/operations/file_ops.py` | Worker-side file ops |
| `…/blitzy_utils/runner_ops/operations/git_ops.py` | Worker-side git ops |
| `…/blitzy_utils/runner_ops/operations/scm_ops.py` | Worker-side SCM ops including `download_repository` |
| `…/blitzy_utils/runner_ops/__main__.py` | `python -m blitzy_utils.runner_ops <op>` CLI |
| `…/blitzy_utils/runner_ops/consts.py` | `BLITZY_RUNNER_READY_TIMEOUT` |
| `…/blitzy_utils/scm.py:228 download_repository_to_disk` | The clone primitive |
| `…/blitzy_utils/disk.py:101 get_disk_path` | Where clones land |

## blitzy_platform_shared (sibling repo archie-shared)

| File | Role |
|---|---|
| `/Users/m.makhzomi/Projects/blitzy-platform/archie-shared/blitzy_platform_shared/common/bash.py:217 BashSession` | Subprocess bash with sentinel-based result capture |
| `…/common/bash.py:1020 BashSessionManager` | Multi-session coordinator |
| `…/common/bash.py:1109 routed_restart_bash_session` | Bridge to RunnerSession |
| `…/common/utils.py:read_range` | Line-range reader (used by `download_file`) |
| `…/common/consts.py:GITHUB_FILE_RETRIEVAL_ERROR` | Tool error sentinel |

## archie-job-reverse-code-generator (image reference)

| File | Role |
|---|---|
| `/Users/m.makhzomi/Projects/blitzy-platform/archie-job-reverse-code-generator/Dockerfile` | Ubuntu 24.04 + Python 3.12 + git + git-lfs + Node 20 + Chrome + Docker. ~3 GB. **Too heavy for chat — reject for chat-worker base.** |
| `…/main.py:81 generate_reverse_code` | Pulls `EVENT_DATA`, instantiates `RunnerSession` if `should_use_runner()` |
| `…/lib/blitzy/helper.py:30,693 routed_download_repository_to_disk` | Reference for "how to use the runner from a job" |
