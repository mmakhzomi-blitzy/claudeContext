# Per-chat-session worker provisioning for archie-service-chat — Plan

**Prepared:** 2026-05-06
**Author:** mohammed@blitzy.com
**Scope:** `archie-service-chat`, `archie-client-mediator`, `archie-client-worker`,
`archie-helm-chart`, `blitzy_platform_shared` (sibling repo
`archie-shared/blitzy_platform_shared`), `blitzy_utils` (sibling repo
`blitzy-utils-python/blitzy_utils`), and the `archie-job-reverse-code-generator`
image. **Companion docs in this directory:**
- `repo-inventory.md` — file/path map across all five repos
- `mediator-endpoint-spec.md` — full request/response schema for the new endpoint
- `helm-changes.md` — helm chart deltas (values, RBAC, image, TTL)
- `runner-ops-inventory.md` — every existing runner op + ops still missing for chat
- `reverse-code-generator-image.md` — extracted Dockerfile contents for image strategy

---

## 1. Current state — what already exists

### 1.1 The "runner" abstraction is **already built**, not used by chat yet

The user's prompt asked for "worker-aware versions of file/git ops in `blitzy-util`."
Those already exist in `blitzy_utils.runner_ops`. The reverse-code-generator job
already provisions a per-job worker through the mediator and routes file/git ops
to it. The chat service is the only major Python service in the platform that
**does not** use this path today.

| Surface | Where | What exists |
|---|---|---|
| Per-job worker session | `blitzy-utils-python/blitzy_utils/blitzy_utils/runner_ops/session.py:21` | `RunnerSession` — `start()` calls mediator `POST /api/v1/runners`, `stop()` calls `DELETE`, `run(op, **kw)` submits a `python -m blitzy_utils.runner_ops <op>` command and polls for terminal status |
| Default mediator client | `blitzy-utils-python/blitzy_utils/blitzy_utils/blitzy_client.py:232` | `BlitzyClient.create_runner` / `delete_runner` / `create_job_command` / `get_command_status` / `restart_session` / `get_runner_status` / `wait_for_ready`. Talks to relay → mediator via `SERVICE_URL_RELAY` and an `X-Client-ID` header |
| Mediator endpoints | `archie-client-mediator/src/api/routes/runners.py:74` | `POST /runners`, `DELETE /runners/<job_id>`, `GET /runners/<job_id>` (status). All take `RunnerConfig` (`cpu_request`, `cpu_limit`, `memory_request`, `memory_limit`, `image_override`, `environment_variables`, `timeout_seconds`, `replicas`). Calls `RunnerService.create_runner` → `KubernetesService.create_deployment` |
| Mediator k8s service | `archie-client-mediator/src/services/kubernetes_service.py:285` | `create_deployment` builds a `V1Deployment` with `WORKER_IMAGE` env var, optional `image_override`, env from `EVENT_DATA`, `envFrom` ConfigMap (`COMMON_CONFIGMAP_NAME`), pull secret, SA `WORKER_SERVICE_ACCOUNT` |
| Mediator consts | `archie-client-mediator/src/consts.py:33` | `WORKER_IMAGE`, `WORKER_CPU_REQUEST`, `WORKER_MEMORY_LIMIT`, `K8S_NAMESPACE`, `WORKER_SERVICE_ACCOUNT`, `IMAGE_PULL_SECRET_NAME` |
| Worker | `archie-client-worker/main.py` + `src/blitzy_worker/` | Long-running pod, polls Redis queue (the relay forwards `create_job_command` requests onto the queue), executes commands via `BashSession`. `startup.py:47 CloneRepositoryStep` — **already calls `download_repository_to_disk()` at boot using EVENT_DATA** |
| Worker image (chart) | `archie-helm-chart/blitzy-client/chart/values-dev.yaml:13` | `images.clientWorker.repository=archie-client-worker`, `tag=44b4962…`. Wired into the mediator pod env via `archie-helm-chart/blitzy-client/chart/templates/configmap.yaml:36` (`WORKER_IMAGE: "{{ include "chart.image" ... }}"`) |
| Routed file/git ops | `blitzy-utils-python/blitzy_utils/blitzy_utils/runner_ops/client.py` | `routed_download_repository_to_disk`, `routed_find_closest_ancestor_branch`, `routed_get_changed_files_between_commits`, `routed_count_lines_in_file`, … (~25 functions). Each takes an optional `runner_session=` and falls back to local exec when `should_use_runner()` is False |
| Registered ops on the worker | `blitzy-utils-python/blitzy_utils/blitzy_utils/runner_ops/operations/{file,git,scm}_ops.py` | `download_repository`, `setup_github_branch`, `create_commit`, `head_commit`, `create_prs`, `push_pull`, `commit_diff`, `find_ancestor`, `changed_files`, `read_file`, `write_file`, `list_files`, `read_blitzyignore`, `check_path_type`, `get_size`, `write_binary_file`, `count_lines`, `count_lines_batch`, `count_lines_from_file`, `write_file_from_payload`, `get_all_files_from_cloned_repo` |
| Bash session abstraction | `blitzy_platform_shared/common/bash.py:217` (`BashSession`), `:1020` (`BashSessionManager`) | Already used by `reverse-code-generator`. The class supports parallel sessions per agent and graceful restart. `routed_restart_bash_session` (`bash.py:1109`) bridges to a `RunnerSession` when present |
| Reverse-code-generator base image | `archie-job-reverse-code-generator/Dockerfile:1` (`FROM ubuntu:24.04`) | Ubuntu 24.04 + Python 3.12 + git + git-lfs + Node 20 + Chrome + Docker-in-Docker + Chrome DevTools MCP. Fat image (~3 GB) — full toolchain for compiling user code |

**Implication for the user's request:** the bulk of "worker-aware file/git
ops" infrastructure already exists. The chat-service work is to (a) **adopt
the existing `RunnerSession` pattern**, (b) **add a chat-specific provisioning
flow** with longer TTL, and (c) **route VFS calls through the runner** instead
of (or in addition to) Neo4j+GCS. There is no need to invent a new abstraction
in `blitzy_platform_shared`; the right surface is `blitzy_utils.runner_ops`,
which is what `blitzy-util` resolves to.

### 1.2 What `archie-service-chat` does today (the integration target)

| Aspect | File:line | What happens |
|---|---|---|
| HTTP entry | `main.py:51,665,529,777,806` | Quart app. Routes: `GET /start` (SSE), `GET /resume` (SSE), `POST /stop`, `POST /close_session`, `POST /clear`, `POST /clear-cache`, `POST /ack`, `GET /health` |
| Session lock (per-thread, in Redis) | `main.py:340–356` (`check_session_lock`) | Key `active_session:<thread_id>`, TTL 600s. Refreshed on every request. Only one active session per thread |
| Close-session handler | `main.py:806–834` | Deletes `active_session:<thread_id>` if owned. **Today: no worker teardown — there is no worker** |
| Workflow bootstrap | `chat_server.py:226 bootstrap_workflow_context(context, proj_uid)` | Builds `tools_config` dict that the LangGraph nodes read via `RunnableConfig.configurable`. Pre-fetches search index + git file list in background |
| File reads (graph-tool) | `src/file_tools.py:25` (`download_file` tool) | Calls `blitzy_utils.scm.download_single_file(...)` against GCS-cached git contents — **no local clone today** |
| File reads (VFS) | `src/vfs/filesystem.py:12,178` | `VirtualFilesystem.read_file` calls `download_single_file` per request; tree comes from `_get_cached_file_paths` (Neo4j); summaries from Neo4j; index in `/tmp/vfs_index` |
| Shell tool | `src/vfs/shell.py:578` (`shell` tool) | Async, in-process pipeline executor over the VFS. **Not a real shell** — only `ls/find/cat/head/tail/grep/wc/cd/pwd/sed/sort/uniq/xargs/jq/echo/seq/summary/search` |
| Tools registered | `chat_server.py:501 platform_tools` + `:528 explorer_tools` | `download_file`, `search_files`, `search_classes`, `search_functions`, ..., `shell` |
| Redis (already used) | `chat_server.py` (`redis_sync_client`, `redis_client`) + `services/access_control.py` | `AsyncRedisSaver` LangGraph checkpointer + ad-hoc keys: `workflow_running:<thread_id>`, `context:<workflow_id>`, `break_agent:<workflow_id>`, `tech_spec:<thread_id>`, `proj_uid:<workflow_id>`, `vfs:tmp:<workflow_id>:*`, `vfs:cwd:<workflow_id>` |
| Stop / cancel | `chat_server.py:2309 stop_agent` | Sets `break_agent:<workflow_id>=true` in Redis; the running graph polls and exits |
| Helm | `archie-helm-chart/helm-chart/apps/archie-service-chat/values.yaml` | Deployment-only chart. No runner-related config today |

**Critical observation:** today the chat service **never clones the repo**. All
file content is fetched on demand via `download_single_file` (GCS-cached); all
file *tree* knowledge comes from Neo4j. Adding a per-session worker therefore
introduces a new mode where a real cloned working tree exists and where
`shell`, `download_file`, and the VFS have a true filesystem to read from.

### 1.3 Reverse-code-generator image — what's in it

(See `reverse-code-generator-image.md` for the full Dockerfile.) Highlights
relevant to chat-worker reuse:

- `FROM ubuntu:24.04` (not `python:slim` like `archie-client-worker`)
- Python 3.12, git + git-lfs 3.7.1, Node 20 LTS + npm 11, Google Chrome stable
- Docker-in-Docker (28.x), supervisord, fuse-overlayfs, iptablesL
- Pre-installed `chrome-devtools-mcp` globally
- Custom `start.sh` that starts `dockerd` (overlay2 → vfs fallback) before exec'ing `python main.py`

Only the toolchain layers are useful for chat. The `start.sh` Docker bootstrap
is reverse-code-generator-specific (it builds and runs the user's project).
**Recommendation:** new `archie-chat-worker` image `FROM` the reverse-code-generator
base layers (or a refactored `archie-base-toolchain` image) but with a different
entrypoint (the existing `archie-client-worker` `BlitzyWorker` poll loop). See
phase 2 for the image-strategy decision.

---

## 2. Proposed architecture

```
                            ┌─────────────────────────────────────┐
                            │   User browser (UI / EventSource)    │
                            └─────────────────┬───────────────────┘
                                              │ /start, /resume (SSE)
                                              ▼
                ┌────────────────────────────────────────────────────┐
                │  archie-service-chat   (Quart, qa branch today)    │
                │  ────────────────────────────────────────────────  │
                │  1. /start handler                                 │
                │     └─ ChatWorkerService.lease(context, proj_uid)  │
                │         ├─ Redis GET chat_worker:<thread_id>       │
                │         │   └─ HIT  → resume same RunnerSession    │
                │         │   └─ MISS → POST /chat-runners (mediator)│
                │         │             SET chat_worker key (6h TTL) │
                │         └─ block until status=READY                │
                │  2. graph nodes call                               │
                │     blitzy_utils.runner_ops.routed_*(              │
                │         ..., runner_session=session)               │
                │  3. /close_session handler                         │
                │     └─ ChatWorkerService.release(thread_id)        │
                │         └─ marks Redis key as drainable; mediator  │
                │            TTL reaper deletes after 6h             │
                └────────────────────────────────────────────────────┘
                                              │
                            BlitzyClient HTTP (relay → mediator)
                                              ▼
        ┌────────────────────────────────────────────────────────────┐
        │  archie-client-mediator                                    │
        │  ──────────────────────────────────────────────────────    │
        │  Existing: POST /runners  /runners/<id>  GET /runners/<id> │
        │  NEW:      POST /chat-runners                              │
        │             ├─ idempotent on label session-key=<sha20>     │
        │             ├─ accepts repo_url + branch + commit + auth   │
        │             ├─ k8s Deployment (image: archie-chat-worker)  │
        │             │   - env CHAT_WORKER_MODE=true                │
        │             │   - env EVENT_DATA={repo_name, branch_name,  │
        │             │       user_id, git_project_repo_id, ...}     │
        │             │   - LABEL chat-worker=true                   │
        │             │   - LABEL chat-worker.blitzy.io/             │
        │             │       session-key=<sha20(thread_id)>         │
        │             │   - ANNOTATION chat-worker.blitzy.io/        │
        │             │       expires-at=<epoch>  (mutable for TTL)  │
        │             └─ returns runner_id, queue, ready handle      │
        │  NEW:      DELETE /chat-runners/<chat_session_key>         │
        │  NEW:      PATCH  /chat-runners/<id>/lease   (extend TTL)  │
        │  NEW:      Background TTL reaper                           │
        │             ├─ lists Deployments with -l chat-worker=true  │
        │             └─ deletes when expires-at annotation < now    │
        └────────────────────────────────────────────────────────────┘
                                              │  k8s API
                                              ▼
        ┌────────────────────────────────────────────────────────────┐
        │  Per-session worker pod (archie-chat-worker:<sha>)         │
        │  ──────────────────────────────────────────────────────    │
        │  - WorkerStartupBuilder(.with_repo_clone()).build().run()  │
        │  - Boot: clone user repo (already implemented in           │
        │    archie-client-worker/src/blitzy_worker/startup.py:47)   │
        │  - Loop: pull command from Redis queue, execute via        │
        │    BashSession, post result back via Redis hash            │
        │  - Toolchain inherited from reverse-code-generator base    │
        └────────────────────────────────────────────────────────────┘
```

**Worker addressing.** The chat service **never speaks to the worker pod
directly.** All traffic goes through the mediator's existing relay+queue
fabric (the mediator owns the WebSocket back to each pod and dispatches
commands by `runner_id`). This is the same pattern reverse-code-generator
already uses; no service-per-pod, no ingress, no NetworkPolicy changes.
Worker-side state lives in the pod's local filesystem (the clone) and per-job
Redis keys.

**Session ↔ worker mapping.** Stored in Redis under
`chat_worker:<thread_id>` with a JSON value:
```json
{
  "runner_id": "chat-<sha256(thread_id)[0:20]>",
  "expires_at": 1746500000,
  "queue_name": "blitzy-queue-chat-…",
  "status": "active"
}
```
- Set by `ChatWorkerService.lease()` on `/start` and refreshed on every `/resume` (sliding 6h TTL).
- Marked `status=draining` by `/close_session`. The Redis TTL is **6h**, but the chat service does not delete the worker — the mediator's TTL reaper does, so a returning user can `lease()` again within the window and skip provisioning.
- Keyed on `thread_id` (the existing `context.to_lg_thread_id()` value). This is naturally per-(company × team × user × project × branch × tech-spec/commit) and survives across browser sessions.

**Coexistence with today's path.** Two layers, in priority order:

1. **Per-tenant gate** (admin-driven, source of truth) — `lease()` calls `BlitzyClient.get_client_type(company_id)` (cached 5 min). Returns `None` for tenants not yet onboarded to a mediator → `lease()` returns `None` and all `routed_*` calls fall back to legacy execution (the `runner_session=None` branch the runner-ops library already supports). This is the **primary** mechanism — it cleanly isolates which tenants are on the new path without code changes per rollout step.
2. **Per-deployment kill switch** (env var) — `CHAT_WORKER_PROVISIONING_ENABLED=true|false`. When `false`, all leases short-circuit to `None` regardless of tenant state. Used for incidents (mediator down across the fleet, k8s outage, etc.). Default `true` per env after phase 1 verification.

Per-repo compatibility (`assert_compatible(client_type, installation_type)`)
is a **third layer** but it's a hard error, not a fallback — if a tenant is
onboarded but the specific repo's SCM type doesn't match their mediator, the
session fails fast with a structured 503. See §3.2.0.

**Migration end-state (phase 6):** the per-tenant `None` branch and the
kill-switch env var both go away. Every session has a worker; the legacy
GCS/Neo4j path is deleted.

---

## 3. Component-by-component plan

### 3.1 `archie-service-chat`

**Scope clarification.** The worker is a **remote filesystem proxy** — it
serves file/git/bash ops for repos the chat service can't reach directly.
The agent loop (LangGraph, prompts, LLM calls, tool dispatch, planning)
**never moves to the worker.** Where things run:

| Concern | Where | Why |
|---|---|---|
| LangGraph agent loop, prompts, tool dispatch | **archie-service-chat** | Always; never the worker |
| File reads (`download_file`, VFS reads) | Worker if leased, else GCS/Neo4j | Worker has the cloned tree |
| Git ops (clone, log, diff) | Worker if leased, else `SERVICE_URL_GITHUB` | Worker has on-prem network access |
| Bash / shell tool | Worker if leased, else current fake-VFS shell | Real bash needs a real FS |

**New module: `src/services/chat_worker_service.py`** — owns gating, the
session→worker mapping, and the `RunnerSession` lifecycle.

```python
class ChatWorkerService:
    def __init__(self, redis_client, blitzy_client: BlitzyClient): ...
    async def lease(self, context: ChatContext) -> RunnerSession | None: ...
    async def release(self, context: ChatContext) -> None: ...  # /close_session
    async def extend_lease(self, context: ChatContext) -> None: ...  # heartbeat
    async def force_terminate(self, context: ChatContext) -> None: ...  # /clear
    def _runner_id_for(self, context) -> str: ...               # deterministic
```

`lease()` algorithm:

1. **Per-tenant gate.** `client_type = blitzy_client.get_client_type(company_id)` (cached 5 min). If `None` → return `None`. Customer is not yet onboarded to a mediator — chat falls through to the **legacy GCS/Neo4j path** (no regression). This branch is removed in the final phase once every customer is on a mediator.
2. **Per-repo compatibility.** `repo = get_git_repo_by_git_project_repo_id(context.git_project_repo_id)` (cached). Call `assert_compatible(client_type, repo.installation_type)` (defined in `blitzy_utils` — see §3.2). On mismatch (e.g. GHES via BLITZY_SHARED) **raise** — never silently fall back. Chat surfaces 503 with `error_code=CHAT_WORKER_INCOMPATIBLE`.
3. **Kill switch.** If env `CHAT_WORKER_PROVISIONING_ENABLED=false` → return `None`. Operator override for incidents; not the primary gate.
4. **Compute identity.** `chat_session_key = sha256(thread_id)[:20]`. `runner_id = "chat-" + chat_session_key`.
5. **Provision (idempotent).** Call `blitzy_client.create_chat_runner(company_id, chat_session_key, runner_id, repo=RepoSpec(...), ttl_seconds=21600)`. The mediator looks up by k8s label `chat-worker.blitzy.io/session-key=<key>` — returns the existing pod with `idempotent_hit=true` if present and `expires_at` is in the future, else creates a new one. **The mediator owns the TTL annotation; chat does not race with it.**
6. **Persist resume hint.** `SET chat_worker:<thread_id>` with `expires_at` from the response. This is a **cache for fast resume**, not the source of truth; if Redis is wiped, the mediator's labels still find the pod.
7. **Block on `wait_for_ready(...)`.** Phase 1 blocks; later phases may stream a "Cloning…" status event (deferred — see §5).
8. **Start heartbeat.** Background asyncio task PATCHes `/chat-runners/<id>/lease` every `HEARTBEAT_INTERVAL=300s` setting `expires_at = now + 6h`. Sliding window — worker stays alive while the chat is active.
9. **Return `RunnerSession`.**

`release()` algorithm (called from `/close_session`):
1. Stop the heartbeat task.
2. PATCH `/chat-runners/<id>/lease` once more with `expires_at = now + 6h`. This starts the 6h reconnect grace period.
3. Update Redis hint with `status=draining`.
4. **Do not** `DELETE` — the mediator's reaper handles teardown when the lease elapses. A reconnect within 6h re-leases the same pod (step 5 above is idempotent, step 8 restarts the heartbeat).

`force_terminate()` (called from `/clear`):
1. `DELETE /chat-runners/<chat_session_key>` — immediate teardown.
2. Delete Redis hint key.

**Failure handling.** If chat-service crashes during an active session, the heartbeat stops; the mediator reaps the pod when `expires_at` elapses (~5–10 min after crash). Reconnect cold-starts a new worker. No orphans.

**Wire-up edits:**
- `main.py:665` (`start_chat`) — after context validation, call `worker = await chat_worker_service.lease(context, proj_uid, repo_meta)`. Thread `worker` into `tools_config` via `bootstrap_workflow_context`.
- `chat_server.py:226 bootstrap_workflow_context` — accept and store `runner_session=worker`. Already stores other handles like `redis_client`.
- `chat_server.py:501 platform_tools` — adopt the `runner_session` from `config.configurable` inside each tool implementation (see 3.2 — most edits land in `blitzy_platform_shared`/`blitzy_utils`, not here).
- `main.py:806 close_session` — call `chat_worker_service.release(context)` before deleting the active-session key.
- `main.py:879 clear_chat_history` — also call `release`; the user explicitly wiped state.
- `main.py:777 stop_workflow` — does **not** terminate the worker (cancel ≠ end of session). Just sets `break_agent`.

**Redis keys added:**
| Key | Value | TTL | Owner |
|---|---|---|---|
| `chat_worker:<thread_id>` | JSON `{runner_id, expires_at, queue_name, status}` | 21600s (6h) | `ChatWorkerService` |
| `chat_worker:reverse:<runner_id>` | `<thread_id>` | 21600s | `ChatWorkerService` (lookup for reaper diagnostics) |

**Files to add/touch:**
- NEW `archie-service-chat/src/services/chat_worker_service.py`
- EDIT `archie-service-chat/src/chat_server.py` (`bootstrap_workflow_context` — accept `runner_session`; `platform_tools` — none directly, but wrap `shell` and `download_file` to pull `runner_session` from config)
- EDIT `archie-service-chat/main.py` (route handlers: `/start`, `/resume`, `/close_session`, `/clear`, plus a new `before_serving` hook to bind `BlitzyClient`)
- EDIT `archie-service-chat/src/consts.py` (new env vars: `CHAT_WORKER_PROVISIONING_ENABLED`, `CHAT_WORKER_TTL_SECONDS=21600`, `CHAT_WORKER_READY_TIMEOUT=300`)
- EDIT `archie-service-chat/requirements.txt` (no change — `blitzy_utils` already pulled in)
- EDIT `archie-service-chat/env_config/env-{dev,qa}.yaml` (add `SERVICE_URL_RELAY`, `CHAT_WORKER_PROVISIONING_ENABLED`)

### 3.2 `blitzy_utils` (sibling repo `blitzy-utils-python/blitzy_utils`)

#### 3.2.0 `assert_compatible(client_type, installation_type)` — shared compatibility matrix

A new helper used by both **chat** (gates `lease()`) and the **mediator**
(rejects mismatched `POST /chat-runners` calls before they hit k8s). Lives
in `blitzy_utils` so there is exactly one matrix.

```python
# blitzy_utils/runner_ops/compatibility.py (new)
#
# The canonical SCM enum lives at blitzy_utils.consts.SvcType. Today's
# values (consts.py:140-144):
#   GITHUB, GITHUB_ENTERPRISE_SERVER, AZURE_DEVOPS, GITLAB, GITLAB_SELF_HOSTED.
# Bitbucket is not supported. AZURE_DEVOPS covers BOTH cloud and self-hosted —
# the distinction is detected at runtime by URL via
# blitzy_utils.azure.is_self_hosted_azure_devops(url). See §3.2.0 caveat below.
#
# Default compatibility matrix. Cloud-only SCM types work on either mediator;
# self-hosted/enterprise variants require CUSTOMER_DEDICATED.
#
# Override at runtime via env var CHAT_WORKER_COMPAT_OVERRIDES (JSON).

from blitzy_utils.consts import SvcType
from blitzy_utils.enums import ClientInstallationType

_MATRIX: dict[ClientInstallationType, set[SvcType]] = {
    ClientInstallationType.BLITZY_SHARED: {
        SvcType.GITHUB,
        SvcType.GITLAB,
        # AZURE_DEVOPS handled specially — see _is_compatible_ado below.
    },
    ClientInstallationType.CUSTOMER_DEDICATED: {
        SvcType.GITHUB,
        SvcType.GITHUB_ENTERPRISE_SERVER,
        SvcType.GITLAB,
        SvcType.GITLAB_SELF_HOSTED,
        SvcType.AZURE_DEVOPS,  # covers cloud + self-hosted
    },
}

def assert_compatible(
    client_type: ClientInstallationType,
    installation_type: SvcType,
    repo_url: str | None = None,  # required only for AZURE_DEVOPS
) -> None:
    """Raise IncompatibleClientInstallation if the pair isn't supported.
    Used by both archie-service-chat (lease gate) and archie-client-mediator
    (POST /chat-runners validation). Single source of truth."""
    if installation_type == SvcType.AZURE_DEVOPS:
        if not _is_compatible_ado(client_type, repo_url):
            raise IncompatibleClientInstallation(client_type, installation_type, repo_url)
        return
    allowed = _MATRIX.get(client_type, set())
    if installation_type not in allowed:
        raise IncompatibleClientInstallation(client_type, installation_type)

def _is_compatible_ado(client_type: ClientInstallationType, repo_url: str | None) -> bool:
    if client_type == ClientInstallationType.CUSTOMER_DEDICATED:
        return True  # ADO of either flavour OK on customer mediator
    # BLITZY_SHARED: only ADO Cloud (dev.azure.com), not on-prem ADO Server.
    from blitzy_utils.azure import is_self_hosted_azure_devops
    if repo_url is None:
        # Fail closed when we can't tell — caller must pass the URL.
        return False
    return not is_self_hosted_azure_devops(repo_url)
```

**The AZURE_DEVOPS caveat.** `SvcType.AZURE_DEVOPS` is a single enum value
covering both cloud (`dev.azure.com`) and self-hosted ADO Server. The
cloud/self-hosted distinction is made by URL inspection
(`blitzy_utils.azure.is_self_hosted_azure_devops` at `azure.py:5163`).
That means `assert_compatible` needs the repo URL passed in, not just
the `installation_type`. Both call sites (chat's `lease()` and the
mediator's `POST /chat-runners` validation) already have the URL —
chat from `RepoSpec.server`, mediator from the request body. Update §3.1
and §3.4 to pass it through.

**Why a constant module-level dict instead of admin API?** Latency: chat is
on the request path. The matrix changes rarely (new SCM support, not per
customer). Override knob (`CHAT_WORKER_COMPAT_OVERRIDES` env var) is enough
for hotfixes; admin-API-driven dispatch can come later if the matrix ever
becomes per-tenant.

**SCM types not yet supported** — Bitbucket has no `SvcType` entry today.
If/when added, append the new value to the relevant `_MATRIX` row(s).
**§5 question #1 is now closed by reading `consts.py` directly** — the
canonical strings above are the production values.

#### 3.2.1 Existing inventory of runner ops

**Inventory of operations the chat service needs that already exist:** see
`runner-ops-inventory.md`. Summary: clone (`download_repository`), tree
(`get_all_files_from_cloned_repo`), file read (`read_file`), file write
(`write_file`), exists/type/size, line counting, blitzyignore. Nothing is
*missing* for read-only use.

**What needs to be added to `blitzy_utils.runner_ops`:**

| New op | Why chat needs it | Lives at |
|---|---|---|
| `runner_op("ls_dir")` | VFS `list_dir` — directory listing with `[{name,type}]` shape | `blitzy_utils/runner_ops/operations/file_ops.py` |
| `runner_op("glob_match")` | VFS `glob_match` — `fnmatch` against tree | same |
| `runner_op("grep_files")` | `grep -rn` over a path-set with regex; returns hits | same (or `text_ops.py` new module) |
| `runner_op("find_files")` | `find` semantics with `-name` / `-type` filters | same |
| `runner_op("cat_with_range")` | line-range read (today: `read_range` post-fetch). Saves a round-trip on multi-line reads | same |
| `runner_op("git_status")` | future write-mode safety | `git_ops.py` |

These are 30–80 line each; all delegate to existing helpers in
`blitzy_utils.git_helpers` or `blitzy_utils.scm`. Bake into the worker image
via the existing `pip install blitzy_utils` step in `archie-client-worker/Dockerfile`
(rebuilt automatically when blitzy_utils is bumped — see `requirements.txt`).

**Client-side wrappers** (one per op) added in
`blitzy_utils/runner_ops/client.py` following the existing `routed_*` pattern.

### 3.3 `blitzy_platform_shared` (sibling `archie-shared/blitzy_platform_shared`)

The user mentioned `blitzy-util` "where worker-aware versions of those operations
need to live, mirroring the existing local versions." The natural home is
`blitzy_utils.runner_ops` (3.2). However, the chat-only **VFS adapter layer** —
which decides whether to read from Neo4j+GCS (today) or from a worker (new) —
is a chat-aware concern that doesn't belong in `blitzy_utils`. It does not
belong in `blitzy_platform_shared` either (which is shared with the jobs).

**Recommendation:** keep the dispatch inside `archie-service-chat/src/vfs/`.
Refactor `VirtualFilesystem` so its content-fetching methods (`read_file`,
`get_summary`, `read_files_batch`, the glob/find paths in `commands.py`)
delegate to a new `FileBackend` interface with two implementations:
- `Neo4jGcsBackend` — current behaviour, lifted out of `filesystem.py`.
- `RunnerBackend` — wraps a `RunnerSession`; calls `routed_read_file`, `routed_get_all_files_from_cloned_repo`, `routed_grep_files`, etc.

Backend selection happens in `bootstrap_workflow_context` based on
`tools_config.get("runner_session")`.

**Files to add/touch:**
- NEW `archie-service-chat/src/vfs/backends/__init__.py`
- NEW `archie-service-chat/src/vfs/backends/base.py` (interface)
- NEW `archie-service-chat/src/vfs/backends/neo4j_gcs.py` (lifted from current `filesystem.py`)
- NEW `archie-service-chat/src/vfs/backends/runner.py`
- EDIT `archie-service-chat/src/vfs/filesystem.py` (constructor takes `backend: FileBackend`)
- EDIT `archie-service-chat/src/vfs/shell.py:617` (read `runner_session` from config and pass into `VirtualFilesystem`)

### 3.4 `archie-client-mediator`

#### 3.4.0 Design fork: how does `EVENT_DATA` reach the worker?

The existing `POST /runners` endpoint **does not accept `EVENT_DATA` from the
caller**. The mediator fetches it server-side by calling
`BlitzyService.get_job_info(job_id)` inside `expose_environments()` — see the
existing flow at `archie-client-mediator/src/services/kubernetes_service.py:180-183`
(injects `EVENT_DATA`) and the upstream lookup that produces it. The job
record (with `repo_name`, `branch_name`, `user_id`, `git_project_repo_id`,
`head_commit_hash`) lives in the **archie-service-backend** database keyed
by `job_id`.

This forces an architectural choice for chat:

**Option A — Mint a synthetic "chat job" in archie-service-backend at session
start.** Chat calls a new backend endpoint to create a `Job`-like record for
the session, gets back a `job_id`, then calls the existing `POST /runners`
with that id. Reuses 100% of the existing fetch path on the mediator.

- Pros: mediator code path is unchanged; worker boot path is unchanged;
  consistent with how every other service in the platform already requests
  workers; auditable in the same DB the rest of the platform uses.
- Cons: requires a new endpoint and DB migration on archie-service-backend;
  one DB row per chat session (cheap but accumulates); the chat session id
  and the job id are now two separate things to keep in sync.

**Option B — New `POST /chat-runners` endpoint that accepts `event_data`
inline in the request body.** Chat passes `repo_name`/`branch_name`/etc.
directly. Mediator skips `get_job_info` and writes the body straight into
the pod's `EVENT_DATA` env var.

- Pros: no archie-service-backend changes; chat owns the metadata it sends;
  one fewer round-trip; the chat session id IS the runner identity (no
  parallel id to reconcile).
- Cons: forks the mediator's worker-creation code (now two paths to
  maintain); chat has to source `git_project_repo_id` and an installation
  token from somewhere (today reverse-code-generator gets these via the
  Job record).

**Current proposal: Option B**, on the grounds that chat already has the
repo metadata in `ChatContext` (no new lookup needed) and the mediator's
`_create_runner_inner` refactor below already factors out the shared k8s
machinery so the divergence is small. **Open question #11** in §5 asks the
user to confirm before phase 3.

#### 3.4.1 Label/annotation rule (k8s constraint)

The new endpoint splits metadata across labels and annotations along k8s
semantics — not arbitrarily:

| Use case | Goes in | Why |
|---|---|---|
| Identifying the chat-worker fleet (`chat-worker=true`) | **Label** | Labels are query-indexed; the reaper does `kubectl get deploy -l chat-worker=true` |
| Idempotency lookup by chat session (`chat-worker.blitzy.io/session-key=<sha20(thread_id)>`) | **Label** | Has to be queryable: `kubectl get deploy -l chat-worker.blitzy.io/session-key=<key>` |
| TTL deadline (`chat-worker.blitzy.io/expires-at=<epoch>`) | **Annotation** | Labels are immutable on a running pod; annotations can be patched. Every TTL extension must mutate this value, so it can't be a label. |
| Free-form chat metadata (tech_spec_id, agent name, etc.) | **Annotation** | Not used for filtering; arbitrary length is fine |

This is a hard k8s constraint, not a style choice — labels you can't `PATCH`
on a running pod (you'd have to delete + recreate, which kills the worker).

#### 3.4.2 Endpoint shape

**`POST /chat-runners`** (full schema in `mediator-endpoint-spec.md`).
Differences from `POST /runners`:
- Idempotent on the `chat-worker.blitzy.io/session-key=<hash>` **label** — looks up via k8s API (single source of truth, not Redis cache).
- Forces image to `CHAT_WORKER_IMAGE` env var (a new mediator config var) — does **not** allow `image_override`.
- Accepts a `repo` block (`repo_name`, `branch_name`, `head_commit_hash`, `git_project_repo_id`, `user_id`, **`installation_type`**) that becomes `EVENT_DATA` (Option B above).
- **Validates compatibility** by calling `blitzy_utils.runner_ops.compatibility.assert_compatible(self.client_type, repo.installation_type)` before any k8s call. This is a defence-in-depth check — chat already gates on the same function in `lease()`. On mismatch returns 400 with `error_code=CHAT_WORKER_INCOMPATIBLE`.
- Applies the labels/annotations split per §3.4.1.
- Initial `expires-at` annotation = `now + ttl_seconds` (default 21600).

**`PATCH /chat-runners/<runner_id>/lease`** — extends the TTL. Body:
`{"ttl_seconds": 21600}`. Mediator updates the `chat-worker.blitzy.io/expires-at`
annotation via a k8s `patch_namespaced_deployment` call. Returns `200`
with the new `expires_at`, or `404` if the deployment is gone (worker
was already reaped — chat treats this as "lease lost," tears down the
session, and starts a fresh one). **This is the only mutable field**;
labels remain immutable. Authorization: requester's `company_id` must
match the runner's `blitzy-job-id` label.

**`DELETE /chat-runners/<chat_session_key>`** — equivalent to
`DELETE /runners/<runner_id>` but takes the chat key instead. Used by
ops and `/clear`. Same `company_id` check as PATCH.

**Background reaper.** A coroutine kicked off in mediator startup that
every 5 min lists Deployments with `chat-worker=true` (label) and deletes
those whose `chat-worker.blitzy.io/expires-at` annotation is in the past.
This is the **authoritative** TTL — chat's heartbeat is the policy that
sets the value, but the reaper enforces. Survives chat-service crash.

**Files to add/touch:**
- NEW `archie-client-mediator/src/api/routes/chat_runners.py` (POST, PATCH lease, DELETE)
- EDIT `archie-client-mediator/src/api/models.py` (new pydantic models: `CreateChatRunnerRequest`, `CreateChatRunnerResponse`, `ExtendLeaseRequest`, `RepoSpec`)
- EDIT `archie-client-mediator/src/services/runner_service.py` (refactor `create_runner` to share a private `_create_runner_inner`; add `create_chat_runner` with idempotency-by-label and `extend_chat_runner_lease` for PATCH)
- EDIT `archie-client-mediator/src/services/kubernetes_service.py` (accept and apply the `expires-at` annotation; add `list_chat_workers_with_expiry()` for the reaper; add `patch_chat_runner_annotation()` for PATCH)
- NEW `archie-client-mediator/src/workers/chat_worker_reaper.py`
- EDIT `archie-client-mediator/src/consts.py` (`CHAT_WORKER_IMAGE`, `CHAT_WORKER_TTL_SECONDS=21600`, `CHAT_WORKER_REAPER_INTERVAL_SECONDS=300`, `CHAT_WORKER_HEARTBEAT_INTERVAL_SECONDS=300`)
- EDIT `archie-client-mediator/main.py` (start the reaper task)
- EDIT `archie-client-mediator/swagger.yaml` (regenerate models — same pipeline that produced `src/api/models.py:1-3` "generated by datamodel-codegen")

### 3.5 `archie-client-worker`

The worker already implements clone-on-boot
(`src/blitzy_worker/startup.py:47 CloneRepositoryStep`) and the polling loop.
**Minimal changes:**
- Recognise the new `CHAT_WORKER_MODE=true` env var. When set, register chat-only ops if any (none right now), tighten log fields (`session_kind=chat`).
- Optional: add an `IDLE_HEARTBEAT_INTERVAL` to publish a heartbeat key the mediator's reaper can use as a tiebreaker (annotation-based TTL is enough for v1; defer this).

**Image strategy (the load-bearing decision):**

Option A — **Reuse `archie-client-worker` image as-is.** Pros: smallest blast
radius, already tested. Cons: missing some toolchain pieces (no Chrome MCP,
no Docker). For the **read-only chat use case** this is fine — the chat agent
needs git + bash + python, all of which `archie-client-worker` has.

Option B — **New `archie-chat-worker` image** that `FROM
archie-client-worker:<sha>` and adds chat-specific layers (e.g. ripgrep, fd,
ast-grep — speeds up search ops). Pros: tunable for chat. Cons: another image
to build.

Option C — **Reuse `archie-job-reverse-code-generator` directly**, switching
its entrypoint. Cons: 3 GB image with Docker-in-Docker, supervisord, Chrome —
huge for a read-only chat session. **Reject.**

**Recommendation: Option A for phase 1**, revisit Option B in phase 4 once
real workloads expose the missing tools. Phase 1 keeps "image strategy" out
of the critical path.

### 3.6 `archie-helm-chart`

(Full deltas in `helm-changes.md`.)

- `blitzy-client/chart/templates/configmap.yaml` — add `CHAT_WORKER_IMAGE`, `CHAT_WORKER_TTL_SECONDS`, `CHAT_WORKER_REAPER_INTERVAL_SECONDS`.
- `blitzy-client/chart/values{,-dev,-qa,-prod}.yaml` — `images.chatWorker` → reuse `archie-client-worker` image (`{repository, tag}` indirection).
- `helm-chart/apps/archie-service-chat/values.yaml` — add `SERVICE_URL_RELAY` ConfigMap entry, `CHAT_WORKER_PROVISIONING_ENABLED` (start `false` per env, flip via PR per env after smoke tests), `BLITZY_CLIENT_MEDIATOR_URL` if direct (currently the chat service is in the platform cluster, mediator is in the **client** cluster → traffic goes through `archie-service-relay`, just like reverse-code-generator).
- Mediator RBAC (`blitzy-client/chart/templates/mediator-sa.yaml`) — verify the SA already has `deployments.create/delete/list/watch` and `pods.list/get`. If `list with annotation filter` doesn't work in the existing role, add `list` on `apps/v1.Deployments` cluster-wide-or-namespaced.

---

## 4. Implementation phases

Phasing has **two axes**: (a) cohort coverage — which `(client_type, installation_type)` pairs are eligible — and (b) capability coverage — which ops route through the worker. Each phase widens one axis; legacy GCS/Neo4j path stays as the fallback for everything not yet migrated, so chat is never regressed.

| Phase | Eligible cohort | Worker-routed ops | Legacy path covers | Exit signal |
|---|---|---|---|---|
| **1** | `CUSTOMER_DEDICATED` × `GITHUB_ENTERPRISE_SERVER` (today's prod) | `clone` + `read_file` (`download_file` tool) | All other tools (tree, search, shell, summary, write) | One real customer's chat sessions show worker-routed reads in Datadog with no regression for 7 days |
| **2** | + all `CUSTOMER_DEDICATED` SCMs: `GITHUB`, `GITLAB`, `GITLAB_SELF_HOSTED`, `AZURE_DEVOPS` (cloud and self-hosted) | same as phase 1 | same | All `SvcType` values verified end-to-end against a sample repo each |
| **3** | + `BLITZY_SHARED` × cloud-only SCMs: `GITHUB`, `GITLAB`, `AZURE_DEVOPS` (cloud, gated on URL — self-hosted ADO routed via CUSTOMER_DEDICATED only) | same | same | Blitzy's shared mediator deployed; flag flipped per-environment |
| **4** | All cohorts above | + tree ops (`list_dir`, `glob_match`), `grep`, `find`, real `bash` | Summary/search-classes (Neo4j-cached, defer) and write ops | `shell "git log --oneline | head"` works on a leased session |
| **5** | Onboarding push: every customer onto a mediator | same | shrunk to summary/search-classes | `get_client_type()` returns non-`None` for every active company |
| **6** | All | All | **Deleted** — `Neo4jGcsBackend`, fake-VFS shell, and `download_single_file` GCS path removed | Code paths gone; chat depends on the worker |

Each phase has its own exit signal, not a fixed week count. Phase 1 is the
load-bearing one; phases 2–3 are mostly configuration; phases 4–6 are
incremental capability + cleanup.

### Phase 1 — minimum viable end-to-end

**Goal:** `CUSTOMER_DEDICATED` + `GITHUB_ENTERPRISE` chat sessions lease a
worker, clone the repo, and route `download_file` through it. Every other
tool keeps its current path. Zero regression for any tenant.

**Deliverables**
- Per-tenant gate (`get_client_type`) + per-repo validation (`assert_compatible`) in `ChatWorkerService.lease()`.
- `BlitzyClient.create_chat_runner()` + new `RunnerSession` initialiser in `blitzy_utils`.
- `POST /chat-runners`, `PATCH /chat-runners/<id>/lease`, `DELETE /chat-runners/<key>`, mediator reaper.
- `download_file` tool routes through `routed_read_file(...)` when `runner_session` is bound; falls back to legacy GCS lookup otherwise.
- Lease + release wired to `/start` and `/close_session`; heartbeat coroutine running.
- Datadog dashboard: lease success rate, HIT vs MISS, ready latency, reads-via-worker count.

**Out of scope for phase 1:** other SCM types (phase 2), `BLITZY_SHARED` (phase 3), tree/grep/bash routing (phase 4), legacy-path retirement (phase 6).

**Files touched**: `archie-service-chat/{main.py, src/services/chat_worker_service.py (new), src/file_tools.py, src/chat_server.py, src/consts.py}`; `archie-client-mediator/src/{api/routes/chat_runners.py (new), services/runner_service.py, services/kubernetes_service.py, workers/chat_worker_reaper.py (new), consts.py, main.py}`; `blitzy-utils-python/blitzy_utils/blitzy_utils/{blitzy_client.py, runner_ops/compatibility.py (new), runner_ops/session.py}`.

### Phase 2 — SCM coverage on `CUSTOMER_DEDICATED`

Mostly data work. Add the remaining `installation_type` values to `_MATRIX`. Test one repo per SCM provider end-to-end. No code changes in chat or mediator beyond compat-matrix entries (and any provider-specific quirks the worker's existing `download_repository_to_disk` already handles).

### Phase 3 — `BLITZY_SHARED` rollout

Provision the shared mediator. Flag-flip per environment after smoke. Compat matrix already supports the cloud SCM rows; this phase exercises them.

### Phase 4 — Capability expansion (read ops + bash)

Add the new runner ops listed in §3.2.1 (`ls_dir`, `glob_match`, `grep_files`, `find_files`, `cat_with_range`). `RunnerBackend` implements the full VFS contract. `src/vfs/shell.py` swaps the in-process pipeline executor for a thin shim over `runner_session.run_bash(...)`. Keep the in-process fallback for sessions without a leased worker (legacy tenants in phases 1–4).

### Phase 5 — Customer onboarding push

Operations / sales work. Get every active company onto a mediator (whichever type fits). When `get_client_type()` returns non-`None` for 100% of active companies for a sustained period, phase 6 is unblocked.

### Phase 6 — Legacy-path retirement

Delete:
- `Neo4jGcsBackend` and the file-content branch of `VirtualFilesystem`
- The legacy `download_single_file` GCS-lookup path in `src/file_tools.py`
- The fake-VFS `shell` pipeline executor in `src/vfs/shell.py`
- The `runner_session is None` fallback branches in every routed op
- The `CHAT_WORKER_PROVISIONING_ENABLED` kill switch (no longer needs an off-state)
- Open question #9 ("`USE_RUNNER` env approach for jobs") becomes moot — chat never used it

This is a substantial code deletion. Plan it as a dedicated phase, not a tail-end cleanup; expect ~2–3 PRs.

### Failure semantics (cross-cutting — applies from phase 1 onward)

Document in `archie-service-chat/docs/CHAT_WORKER.md` and revisit each phase:

- **Mediator unreachable** during a leased session → fail the `/start` with 503. **Do not** silently fall back to the legacy path for tenants whose `client_type != None`; that masks production divergence and confuses debugging. Tenants with `client_type == None` were never going to get a worker, so they continue on legacy as designed.
- **Worker dies mid-session** → next `routed_*` call raises `RunnerOperationError`. Chat catches at the tool layer, logs, and surfaces the existing `GITHUB_FILE_RETRIEVAL_ERROR` sentinel for `download_file` so the agent's existing error handling kicks in.
- **PATCH lease returns 404 / 409** → chat treats as "lease lost," tears down the heartbeat, and provisions a fresh worker on the next user turn (cold-start cost ~30–60 s for re-clone).
- **`assert_compatible` fails** → 503 with structured `error_code=CHAT_WORKER_INCOMPATIBLE`. UI shows a real "this repo isn't supported on your installation" message, not a silent failure.

---

## 5. Open questions

### Answered (frozen — do not re-litigate without explicit user pushback)

| # | Question | Resolution |
|---|---|---|
| A1 | Worker cluster | Client cluster (per-company), routed via existing relay (`SERVICE_URL_RELAY`) — same as reverse-code-generator |
| A2 | Multi-tenant cluster lookup | `BlitzyClient.get_client_url(company_id)` already resolves; reuse |
| A3 | Lease key | `sha256(thread_id)[:20]` — naturally per-(company × team × user × project × branch × commit) |
| A4 | Per-tenant vs per-repo gate | Per-tenant (`get_client_type`) is the gate; per-repo (`installation_type`) is the compatibility check |
| A5 | TTL ownership | Chat owns the policy via heartbeat (PATCH every 5 min, sliding 6h); mediator enforces wall-clock via annotation reaper |
| A6 | Reconnect resets TTL | Naturally via idempotent `POST /chat-runners` returning the same pod and PATCHing `expires_at = now + 6h` |
| A7 | EVENT_DATA sourcing | Option B — new `POST /chat-runners` accepts `event_data` inline; mediator skips `get_job_info` |
| A8 | Image choice | `archie-client-worker` image as-is for phase 1; revisit if chat needs missing toolchain (deferred) |
| A9 | Phase 1 scope | `CUSTOMER_DEDICATED` × `GITHUB_ENTERPRISE` only, with `clone + read_file` worker-routed; everything else stays on legacy path; zero regression for any tenant |
| A10 | Compatibility matrix location | Module-level dict in `blitzy_utils.runner_ops.compatibility`; env-var override (`CHAT_WORKER_COMPAT_OVERRIDES`) for hotfixes; admin-API-driven dispatch deferred |
| A11 | Routing scope | Worker = remote FS proxy only (file/git/bash). Agent loop, prompts, LLM calls always run in `archie-service-chat` |

### Still open (please answer before phase 1)

> **Closed since last revision:** former Q1 ("canonical SCM type strings") — confirmed by reading `blitzy_utils.consts.SvcType` at `consts.py:140-144`. The five values are `GITHUB`, `GITHUB_ENTERPRISE_SERVER`, `AZURE_DEVOPS`, `GITLAB`, `GITLAB_SELF_HOSTED`. No Bitbucket support today. `AZURE_DEVOPS` covers both cloud and self-hosted (URL-discriminated). Matrix in §3.2.0 updated.

1. **Auth for repo clone.** `download_repository_to_disk` requires the GitHub installation token. Today reverse-code-generator passes this via `EVENT_DATA`. Does chat have access to (or the right to mint) the same token at `/start` time, or do we need a new path that mints a per-session credential?
2. **Worker pod resource budget.** Defaults are 500m CPU / 8 GiB memory. Acceptable for an idle chat worker held for 6h? At 100 concurrent chats that's 50 CPU / 800 GiB. Need either lower defaults (e.g. 200m / 1 GiB) or a per-company concurrent cap.
3. **`/clear` semantics.** Should `/clear` delete the worker (force re-clone) or just reset the agent state? Plan currently terminates the worker via `force_terminate()`.
4. **Streaming readiness.** Phase 1 blocks `/start` until the worker is READY (~30–60 s cold start incl. clone). Chat UX cost is high. Should we instead stream a "Cloning your repository…" event and only delay tool calls (not the whole stream) until ready? Defer to phase 4 if not in phase 1's critical path.
5. **Local dev.** Chat service runs locally against stage. Will the chat-worker live in stage's client cluster (local chat just calls stage's mediator), or do we need a `LOCAL_DEVELOPMENT` shortcut that skips provisioning? (Easiest: stage-only; local dev sets `CHAT_WORKER_PROVISIONING_ENABLED=false`.)
6. **Cloud-SCM-but-firewalled.** Customer with a github.com repo that's only reachable from inside their network (IP allowlist)? If yes, `installation_type` alone isn't sufficient — need a per-repo `requires_runner` flag. Or rare enough to defer?

---

## 6. Risks & rollback

| Risk | Mitigation | Rollback |
|---|---|---|
| Worker provisioning latency degrades `/start` p99 | Phase 1 dashboard; concurrent connection budget; pre-warm pool (deferred to a later phase) | Set `CHAT_WORKER_PROVISIONING_ENABLED=false` per env (ConfigMap edit + pod restart, ~2 min) — kill switch flips all tenants back to legacy |
| Worker fleet runs out of room (100 concurrent chats × 8 GiB) | Lower default resources (#6 above); add a per-company cap in `RunnerService.create_chat_runner` | Same flag-flip; existing pods finish their TTL naturally |
| Mediator reaper bug leaks pods | Reaper unit + integration tests in phase 3; alert on Deployment count with `chat-worker=true` exceeding N | `kubectl delete deploy -l chat-worker=true` is safe; chat will re-provision on next `/start` |
| `routed_*` op missing on the worker (version skew) | `RunnerSession.start()` should query worker capabilities and reject if `chat_worker_protocol_version` < expected. Add to phase 4 deliverable | Worker raises `RunnerOperationError`; chat-side flag flip falls back to local execution |
| GitHub clone fails (token expired, repo too big) | `CloneRepositoryStep` raises; mediator marks pod `FAILED`; chat surfaces 503 with structured error | Same as above; user retries with refreshed token |
| Stale clone vs new pushes during 6 h window | Question #3 above; v1 just re-clones on new commit hash | Cache invalidation by deleting `chat_worker:*` keys in Redis |
| Cost overrun (long-tailed idle workers) | TTL is hard-capped server-side; reaper enforces. Per-company cap in mediator. Datadog cost panel | Reduce `CHAT_WORKER_TTL_SECONDS` to 1800 / 600 (env-only change) |
| Chat service bug deletes someone else's worker | Hash-based `runner_id` (`chat-<sha256(thread_id)[:20]>`) is collision-free; `DELETE /chat-runners` validates the requester's `company_id` matches the runner label | Manual k8s redeploy from helm; mediator rejects mismatched deletes |

---

## 7. Success criteria

- `/start` provisions a per-session worker with the user's repo cloned and routes file/git ops to it within 60 s p95 on cold start, < 3 s on warm reuse.
- 6 h TTL is honoured — pods deleted within 5 min of expiry by the reaper.
- Returning user inside the window resumes the same pod (HIT logged).
- Zero impact on the existing chat workflow when the flag is off.
- Per-env rollout flag-flipped without code redeploy.
