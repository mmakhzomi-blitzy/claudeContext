# Integration Tasks — BashSession Replacement & Worker Repo Cloning

> **Reference docs**:
> - `ARCHITECTURE_REFERENCE.md` — system overview
> - `STORE_AND_FORWARD.md` — relay queue, outbound queue, chunking, multi-worker correlation
> - `CDP_TUNNEL_PLAN.md` — CDP tunnel architecture, resilience, as-built
> - `WORKER_RQ_MIGRATION.md` — SimpleWorker, job lifecycle, failure handling
> - `BASH_SESSION_INTEGRATION_ANALYSIS.md` — BashSession replacement details
> - `DOWNLOAD_REPO_INTEGRATION.md` — worker-side repo cloning
> - `CODE_DOWNLOADER_ANALYSIS.md` — archie-job-code-downloader assessment

---

## Task Status Legend

- [ ] Not started
- [x] Completed

---

## 1. Shared Library: `blitzy-client-utils`

Extract `HttpClient` from mediator into a shared library usable by both mediator and worker.

- [x] Create `blitzy_client_utils/` package under `blitzy-utils-python`
- [x] Extract `HttpClient` with DI constructor (`base_url`, `api_key` + env var fallback)
- [x] Add `pyproject.toml`, `Makefile`, `README.md`
- [x] Add to root `Makefile` UTILS list
- [x] Update mediator to import from `client_utils.http_client`
- [x] Remove `src/utils/http_client.py` from mediator
- [x] Remove `requests` from mediator's direct dependencies

---

## 2. BlitzyClient SDK Enhancements

Extend `BlitzyClient` in `blitzy-utils-python/blitzy_utils` with missing methods
needed for consumer migration.

- [x] Add `restart_session(company_id, job_id)` — wraps `POST /api/v1/jobs/{id}/restart-session`
- [x] Add `execute_command(company_id, job_id, command, ...)` — submit + poll with backoff until terminal status
- [x] Add `execute_bash_tool(company_id, job_id, command, ...)` — execute + format output as `[stdout]\n\n...\n\n[stderr]\n\n...` (drop-in for `handle_bash_tool_response`)
- [x] Add URL caching for `get_client_url()` — singleton pattern, fetch once and store
- [x] Add unit tests for new methods (25/25 passing)

---

## 3. Main Server API Additions

Mirror archie-github-handler SCM APIs on main server under `/v1/client/scm/`,
authenticated via `x-client-key` instead of GCP credentials.

- [x] Implement `GET /v1/client/scm/user/{user_id}/repositories/{git_project_repo_id}/svc_type` — SCM type detection
- [x] Implement `GET /v1/client/scm/github/repositories/{git_project_repo_id}/access-token` — GitHub access token
- [x] Implement `GET /v1/client/scm/repositories/{git_project_repo_id}/secret/access-token` — VCS-agnostic access token
- [x] Verify `GET /v1/client/job/{blitzy_job_id}` returns all fields from `EVENT_DATA` payload

---

## 4. Worker: Repository Download Integration

Enable the worker to clone the repository at startup using `blitzy-utils` + `blitzy-client-utils`.
This is the **foundation for all jobs** — every consumer job needs the code on disk before
executing commands. See `CODE_DOWNLOADER_ANALYSIS.md` for the full assessment.

**Decision**: Only the git clone portion runs on the runner. All file processing, GCS upload,
Pub/Sub notifications, Neo4j graph building stays in GCP (`archie-job-code-downloader` unchanged).

**Design**: Builder pattern for startup pipeline — see `CODE_DOWNLOADER_ANALYSIS.md § Builder Pattern`.

### 4a. Create startup pipeline (`startup.py`)

- [x] Create `archie-client-worker/src/blitzy_worker/startup.py` with:
  - `StartupContext` dataclass — holds `settings`, `event_data` (dict), `working_directory` (str)
  - `StartupStep` ABC — interface with `name` property + `execute(context)` method
  - `CloneRepositoryStep` — calls `download_repository_to_disk()` with fields from `event_data`
  - `WorkerStartupBuilder` — assembles steps via chained methods (`.with_repo_clone()`), returns `WorkerStartup`
  - `WorkerStartup` — executes steps sequentially, logs start/complete per step, fails fast on error

### 4b. Wire startup into `main.py`

- [x] Import `WorkerStartupBuilder` in `archie-client-worker/src/blitzy_worker/main.py`
- [x] After `Settings()` init, before `BlitzyWorker()` creation, run:
  ```python
  startup = (
      WorkerStartupBuilder(settings)
          .with_repo_clone()
          .build()
  )
  startup.execute()
  ```
- [x] `WorkerStartup.execute()` parses `EVENT_DATA` env var (JSON string) into `StartupContext`

### 4c. Dependencies & env vars

- [x] Add `blitzy-client-utils>=0.0.1` to `archie-client-worker/requirements.txt`
- [x] Add `IS_BLITZY_CLIENT_ENV=true` env var to worker pod in `archie-client-mediator/src/services/kubernetes_service.py` `_build_container()` — triggers flag routing in blitzy-utils SCM calls

### 4d. Verification

- [ ] Verify `startup.py` imports resolve correctly
- [ ] Confirm startup runs before BashSession creation (before `BlitzyWorker()`)
- [ ] Confirm `IS_BLITZY_CLIENT_ENV` appears in worker pod env vars
- [ ] End-to-end: mediator creates pod → worker starts → clone repo → BashSession in cloned dir → BLPOP loop ready

---

## 5. Mediator Changes

Update mediator to inject required data into worker pods.

- [x] Inject `EVENT_DATA` (full job metadata JSON from `job_info.job_metadata`) as env var in `kubernetes_service.py` `_build_container()`
- [x] `API_KEY` already available to worker pods via ConfigMap
- [x] Add `RestartSessionResponse` to `swagger.yaml`

---

## 5b. Runner Readiness Signal

Ensure consumers don't send commands until the worker has finished its startup pipeline
(repo clone). Today, K8s deployment `Ready` only means the container is running — the
clone may still be in progress.

**Flow**:
```
Consumer                    Mediator                     Worker
   │                           │                            │
   ├── POST /runners ─────────►│                            │
   │◄── status: DEPLOYING ─────│── create K8s pod ─────────►│
   │                           │                            ├── startup.execute()
   │   poll                    │                            │   (clone repo...)
   ├── GET /runners/{id} ─────►│                            │
   │◄── status: DEPLOYING ─────│                            │
   │                           │                            ├── clone done!
   │   poll                    │  RQ: server_ready job      │
   ├── GET /runners/{id} ─────►│◄──────────────────────────│
   │◄── status: READY ─────────│                            ├── enter BLPOP loop
   │                           │                            │
   ├── POST /commands ────────►│──── LPUSH ────────────────►│ (picks up immediately)
```

### 5b-i. Mediator: readiness signal + status endpoint

- [x] Add `GET /api/v1/runners/{job_id}` — returns runner status (`DEPLOYING`, `READY`, `RUNNING`, etc.)
- [x] Add endpoint to `swagger.yaml`
- [x] Extract named enum schemas from swagger.yaml for human-readable codegen (`RegistrationStatus`, `CreateRunnerStatus`, `DeleteRunnerStatus`, `RunnerStatus`, `CommandPriority`, `CommandSubmissionStatus`, `CommandExecutionStatus`, `SecretsErrorCode`)
- [x] Add `mark_runner_ready()` to `IRunnerService` interface and `RunnerService` implementation
- [x] Add `server_ready_handler.py` RQ worker handler for readiness signals
- [x] Register `server_ready` handler in `job_processor.py`

> **Note**: Readiness is signalled via RQ (`server_ready` job type), not an HTTP endpoint.
> The `POST /runners/{job_id}/ready` endpoint was removed as redundant — the worker
> already communicates with the mediator through RQ.

### 5b-ii. Worker: signal readiness after startup

- [x] After `startup.execute()` succeeds in `main.py`, enqueue `server_ready` job via RQ results queue
- [x] If readiness signal fails, worker exits (no point entering BLPOP without status update)

### 5b-iii. SDK: add `wait_for_ready()` to BlitzyClient

- [x] Add `get_runner_status(company_id, job_id)` — wraps `GET /api/v1/runners/{job_id}`
- [x] Add `wait_for_ready(company_id, job_id, timeout, poll_interval)` — polls `get_runner_status()` with backoff until `READY` or timeout
- [x] Update `create_runner()` or consumer `setup()` to call `wait_for_ready()` after runner creation — done via `RunnerSession.start()` which calls `wait_for_ready()` after `create_runner()` or when status is `DEPLOYING`

---

## 6. Consumer-Side Migration

Replace direct `BashSession` usage with `BlitzyClient` SDK in all three consumer repos.
Changes are identical across all three.

**Repos**:
- `archie-job-reverse-document-generator`
- `archie-job-reverse-code-generator`
- `archie-job-reverse-file-mapper`

**Prerequisites (in `blitzy_utils/runner_ops/`)**:
- [x] Rename `@runner_op` registrations to match original function names (`get_all_files_from_cloned_repo`, `count_lines_in_file`, `find_closest_ancestor_branch`, `get_changed_files_between_commits`)
- [x] Create `client.py` — `remote_` prefixed consumer wrappers with type annotations and `BlitzyGitFile` deserialization
- [x] Add `should_use_runner()` — checks `USE_RUNNER` env var
- [x] Add `routed_` wrappers to `client.py` — same signature as original functions + `runner_session` param; routes to remote or local path based on whether session is provided

**`archie-job-code-downloader`** (routed migration):
- [x] Wire `should_use_runner()` + `RunnerSession` lifecycle in `process_event()`
- [x] Wrap session lifecycle in `try/finally` for auto-cleanup on failure
- [x] Replace flag-branched call sites with `routed_` functions: `routed_download_repository_to_disk`, `routed_find_closest_ancestor_branch`, `routed_get_changed_files_between_commits`, `routed_count_lines_in_file`
- [x] Update `find_reusable_branch` and `download_and_upload` signatures: `use_runner`/`session` → `runner_session`
- [x] Extract `_parse_blitzyignore_content()` shared parser for local/remote paths
- [x] End-to-end verified via Telepresence intercept (runner reuse, all operations, platform event DONE)
- Note: `load_blitzyignore` still flag-branched (local function, not in blitzy-utils)

**`archie-job-code-graph-generator`** (flag-based migration):
- [x] Add `@runner_op("read_file_from_disk")` to `file_ops.py` (worker-side)
- [x] Add `remote_read_file_from_disk()` wrapper to `client.py` (client-side)
- [x] Wire `should_use_runner()` + `RunnerSession` lifecycle in `CodeGraphHelper.__init__` / `setup()` / `teardown()`
- [x] Flag-branch `download_repository_to_disk` in `setup()` (skip when `USE_RUNNER=true`)
- [x] Flag-branch `read_file_from_disk` in `prepare_file()` → `remote_read_file_from_disk()`

**`archie-job-reverse-document-generator`** (flag-based migration):
- [x] Wire `should_use_runner()` + `RunnerSession` lifecycle in `main.py` (create, start, pass to helper, stop in finally)
- [x] Accept `runner_session` param in `ReverseDocumentHelper.__init__`
- [x] Flag-branch `setup()`: skip `download_repository_to_disk` + `restart_bash_session` when runner active
- [x] Flag-branch `gather_context()` bash tool handling: `runner_session.restart_session()` / `run_bash()`
- [x] Flag-branch `process_section()` bash tool handling: same pattern
- [x] `setup()`: use `routed_restart_bash_session()` instead of raw `restart_bash_session()`
- [x] Add `"runner_session"` to `tools_config` in `gather_context()` and `process_section()` for `read_file` tool routing

**`archie-job-reverse-thinking-generator`** (tool-config migration):
- [x] `RunnerSession` lifecycle already wired in `main.py`
- [x] `routed_download_repository_to_disk()` already in `setup()`
- [x] Add `"runner_session"` to `tools_config` in `process_file()` for `read_file` tool routing
- Note: No bash tool in tool lists — no bash branching needed

**Per repo (2 remaining reverse-job consumers)**:
- [x] `archie-job-reverse-code-generator`: routed migration (SCM, bash, file I/O, text editor tools)
- [x] `archie-job-reverse-file-mapper`: routed migration (clone, bash)

**Cross-repo clone integration** (after worker auto-clone removal):
- [x] `archie-job-reverse-document-generator`: replaced flag-branched `download_repository_to_disk` with `routed_download_repository_to_disk`
- [x] `archie-job-reverse-file-mapper`: added `RunnerSession` lifecycle + `routed_download_repository_to_disk`
- [x] `archie-job-code-graph-generator`: already uses `routed_download_repository_to_disk` — no changes needed
- [x] `archie-job-code-downloader`: already uses `routed_download_repository_to_disk` — no changes needed
- [x] Removed `company_id != "default"` guard from RunnerSession creation in reverse-code-generator, reverse-file-mapper, code-graph-generator

**Tool-level runner routing** (archie-shared changes — added 2026-03-30):
- [x] `common/tools.py`: `read_file` tool now uses `routed_read_file_from_disk()` with `runner_session` from `config["configurable"]`
- [x] `common/bash.py`: Added `routed_restart_bash_session()` wrapper — same `(bash_session, is_error)` return signature, routes to runner or local

**Metering / billing runner routing** (CRITICAL — customer billing depends on this):

Metering stats feed into notification payloads that drive customer billing. When runner is active,
files live on the worker pod — any metering that reads the local filesystem will return 0/0.

| Consumer | Metering operation | How it works | Runner-aware? | Notification field(s) |
|----------|-------------------|--------------|---------------|----------------------|
| `code-downloader` | `routed_count_lines_in_file()` (line 526) | Runner op counts lines on worker | **Yes** | `lines_onboarded` |
| `code-downloader` | `ValidationService.validate_onboarding_quota()` (line 210) | `FileSystemRepository` → local `glob` + `open` + line count | **NO — returns 0/0 with runner** | Pre-flight quota check (non-blocking today but billing-critical) |
| `reverse-code-generator` | `routed_get_commit_diff_summary()` (×3 sites) | Runner op runs `git diff` on worker | **Yes** | `files_modified`, `additions`, `edits`, `removals` |
| All others | No filesystem-based metering | N/A | N/A | N/A |

**Fix needed — `ValidationService` quota check** (`archie-shared/metering/`):
- [ ] Create `RunnerFileSystemRepository(IFileSystemRepository)` — implements `find_files()`, `count_lines()`, `is_file()` via runner ops
- [ ] Inject `RunnerFileSystemRepository` into `ValidationService` when runner is active (code-downloader `main.py`)
- [ ] Requires new runner ops on worker: `find_files` (glob), `count_lines` (open + count), `is_file` (os.path.isfile) — or a single composite `get_directory_file_details` runner op

**Bug fixes / improvements**:
- [ ] Add `restart_session()` call in `RunnerSession.start()` when reusing an existing runner (`READY`/`RUNNING` status) — ensures clean bash state for new consumers

**MCP co-location with worker** (Chrome capability — IN PROGRESS):
- [x] Add `RunnerCapability` constants to blitzy-utils (`capabilities.py`)
- [x] Add generic `capabilities` param to `BlitzyClient.create_runner()` and `RunnerSession`
- [x] Add `capabilities_metadata` + `chrome_url` properties to `RunnerSession`
- [x] Add `RunnerCapability` enum to Swagger, `capabilities` + `capabilities_metadata` to API models
- [x] Add `create_chrome_route()` / `delete_chrome_route()` to `KubernetesService` (Service + HTTPRoute)
- [x] Add capability processing to `RunnerService` (create/delete lifecycle, server_metadata persistence)
- [x] Inject `RUNNER_CAPABILITIES` env var into worker pods
- [ ] Worker: Add `StartChromeStep` + `CAPABILITY_STEPS` registry + `with_capability()` to startup builder
- [ ] Worker: Read `RUNNER_CAPABILITIES` in `main.py`, add matching steps
- [ ] Worker: Install Chrome in Dockerfile (runtime stage)
- [ ] Consumer (`reverse-code-generator`): Connect `chrome-devtools-mcp` via `--browserUrl` when remote
- [ ] Consumer (`reverse-code-generator`): `restart_chrome()` via `run_bash` when remote

**Future capabilities** (not yet started):
- [ ] FIGMA capability — co-locate Figma MCP with worker for `reverse-file-mapper`
- ~~IP whitelisting — Envoy SecurityPolicy for Chrome CDP routes~~ (superseded by WebSocket migration — CDP will be tunneled, no public exposure)

---

## 7. Main Server (Blitzy) Integration

Service-layer changes on the main server to support client job tracking, registration, and credential facade.

- [ ] Wire `ClientJobTracking` into the job submission flow — create tracking record when a job is assigned to a client
- [ ] Implement job status update endpoints (ASSIGNED → ACCEPTED → RUNNING → COMPLETED/FAILED)
- [x] Mirror archie-github-handler SCM APIs under `/v1/client/scm/` with API key auth (see Task 3)
- [x] Ensure `GET /v1/client/job/{blitzy_job_id}` returns all fields needed for `EVENT_DATA` payload
- [ ] Implement client registration flow using `ClientInstallation` + `ClientAPIKey` models
- [ ] Add `ClientInstallationAccess` scoping — validate client has access to the project/company resources
- [ ] Add heartbeat tracking — update `last_heartbeat_at` on `ClientInstallation`

---

## 8. Documentation

- [x] `ARCHITECTURE_REFERENCE.md` — system overview, component deep dives, API reference, gaps
- [x] `BASH_SESSION_INTEGRATION_ANALYSIS.md` — restart_bash_session analysis, consumer usage, SDK gaps
- [x] `DOWNLOAD_REPO_INTEGRATION.md` — worker-side repo cloning, main server facade, EVENT_DATA
- [x] `CODE_DOWNLOADER_ANALYSIS.md` — archie-job-code-downloader assessment, Option B decision (clone only on runner)
- [x] `TASKS.md` — this file
- [ ] DB models verification doc (models already exist — no schema changes needed)

---

## Dependency Order

```
1. Shared Library (DONE)
      │
      ├──► 7. Main Server (Blitzy) Integration
      │         │
      │         ├──► 3. Main Server APIs (repo-credentials, job info)
      │         │         │
      │         │         └──► 4. Worker Repo Download
      │         │                   │
      │         │                   └──► 5b. Runner Readiness Signal
      │         │                              │
      │         │                              └──► 6. Consumer-Side Migration
      │         │
      │         └──► 6. Consumer-Side Migration (job tracking must exist)
      │
      ├──► 5. Mediator Changes (EVENT_DATA injection) (DONE)
      │         │
      │         └──► 4. Worker Repo Download
      │
      └──► 2. BlitzyClient SDK Enhancements (DONE)
                │
                ├──► 5b-iii. SDK wait_for_ready()
                │         │
                │         └──► 6. Consumer-Side Migration
                │
                └──► 6. Consumer-Side Migration
```

Critical path: **7 → 3 → 4 → 5b → 6** (main server APIs must exist before worker can clone,
readiness signal must work before consumers can reliably send commands).
Tasks 7 and 5 can proceed in parallel.

---

## 9. WebSocket Migration

> **Reference docs**:
> - `WEBSOCKET_MIGRATION_PLAN.md` — full architecture, components, design decisions
> - `AGENT_ACTION_PLAN.md` — implementation prompt per repository

Replace SDK and CDP communication with a single outbound WebSocket connection via a WS Relay Service. Chrome CDP tunneled through the same connection. Envoy Gateway stays as-is (serves its own purpose).

---

### Repo 1: `archie-service-relay`

**Setup & cleanup:**
- [x] Clean up `consts.py` — remove Google Cloud imports, add Redis config (REDIS_HOST/PORT/USERNAME/PASSWORD)
- [x] Update `requirements.txt` — add flask-socketio, python-socketio, gevent, gevent-websocket, redis
- [x] Update `Dockerfile` — gevent WebSocket worker
- [x] Create `docker-compose.yaml` for local dev (Redis + relay)

**Utils layer:**
- [ ] Create `src/utils/interfaces/i_redis_client_provider.py` — `IRedisClientProvider` ABC
- [ ] Create `src/utils/redis_client_provider.py` — `RedisClientProvider(IRedisClientProvider)`

**Services layer:**
- [ ] Create `src/services/interfaces/i_auth_service.py` — `IAuthService` ABC
- [ ] Create `src/services/auth_service.py` — `AuthService(IAuthService)`, validates API key → returns `client_id`
- [ ] Create `src/services/interfaces/i_registry_service.py` — `IRegistryService` ABC
- [ ] Create `src/services/registry_service.py` — `RegistryService(IRegistryService)`, Redis CRUD for connection state

**Namespaces:**
- [ ] Create `src/namespaces/control.py` — `ControlNamespace`, takes `IAuthService` + `IRegistryService` via constructor
- [ ] Create `src/namespaces/tunnel.py` — `TunnelNamespace`, stateless port forwarding relay

**API routes:**
- [ ] Create `src/api/routes/relay.py` — transparent HTTP→WS proxy, routes by `X-Client-ID` header
- [ ] Create `src/api/routes/connections.py` — `GET /health/connections`
- [ ] Update `swagger.yaml` with relay schemas + run `make pre-setup` to generate models
- [ ] Add `X-Client-ID` to CORS allowed headers
- [ ] Add relay-specific errors to `src/error/errors.py`

**App wiring:**
- [ ] Rewrite `main.py` — Flask + SocketIO init, DI wiring (concrete services → namespaces/routes), namespace registration

---

### Repo 2: `archie-client-mediator`

**Prerequisite — Chrome CDP routing (commit uncommitted changes on `fixing-runner-creation`):**
- [ ] `swagger.yaml` — `RunnerCapability` enum, `capabilities`/`capabilities_metadata` in schemas
- [ ] `src/consts.py` — `GATEWAY_HOST`, `GATEWAY_NAME` constants
- [ ] `src/utils/interfaces/i_kube_client_provider.py` — `get_custom_objects_api()` abstract
- [ ] `src/utils/kube_client_provider.py` — `get_custom_objects_api()` implementation
- [ ] `src/services/interfaces/i_kubernetes_service.py` — `create_chrome_route()`, `delete_chrome_route()` abstracts
- [ ] `src/services/kubernetes_service.py` — Chrome route CRUD, `RUNNER_CAPABILITIES` env injection
- [ ] `src/services/runner_service.py` — capability lifecycle in create/delete/cleanup
- [ ] `src/api/routes/runners.py` — capabilities in responses

**WebSocket migration (Phase 1):**
- [ ] Add `python-socketio[asyncio]` to `requirements.txt`
- [ ] Create `src/services/ws_client.py` — outbound Socket.IO client to relay
- [ ] Create `src/services/ws_router.py` — dispatch WS messages to existing service methods
- [ ] Modify `main.py` — conditional WS client startup (when `MAIN_SERVER_URL` set)
- [ ] Modify `src/services/blitzy_service.py` — skip URL in `register()` when WS client active

**CDP tunnel (Phase 3):**
- [ ] Create `src/services/tunnel_forwarder.py` — forwards tunnel frames to worker Chrome

**Cleanup (Phase 4):**
- [ ] Remove `create_chrome_route()`, `delete_chrome_route()` from `KubernetesService`
- [ ] Remove Chrome route creation/cleanup from `RunnerService`
- [ ] Remove abstract CDP route methods from `IKubernetesService`

---

### Repo 3: `blitzy-utils-python` (SDK)

**Phase 2:**
- [ ] Add `X-Client-ID` header injection to `_RequestsAdapter` in `blitzy_client.py`
- [ ] Add `_client_id` field to `BlitzyClient.__init__`
- [ ] Cache `client_id` from admin response in `get_client_url()`

**Phase 3 — Consumer CDP Proxy:**
- [ ] Create `runner_ops/cdp_proxy.py` — `CDPProxy` class, local proxy on `localhost:9222`
- [ ] Modify consumer helpers to use `CDPProxy` when `chrome_url` exists

---

### Repo 4: `archie-service-admin`

**Phase 2:**
- [ ] Replace `HttpForwardingClient` with `BlitzyClient` in `EnvExchangeService._forward_secrets_to_client()`

**Phase 4:**
- [ ] Remove `src/clients/http_forwarding_client.py`

---

### Repo 5: `archie-helm-chart`

**Phase 1:**
- [ ] Add `/v1/relay` HTTPRoute rule to `api-gateway/values.yaml`
- [ ] Add GCPBackendPolicy with 24-hour WebSocket timeout for relay
- [ ] Create relay service Helm chart (Deployment + ClusterIP Service)

---

### Repo 7: `archie-client-worker`

**Phase 1:**
- [ ] Modify `Dockerfile` — install Google Chrome
- [ ] Create `StartChromeStep` in `startup.py` — headless Chrome on port 9222
- [ ] Add `CAPABILITY_STEPS` registry in `startup.py`
- [ ] Add `with_capability()` to `WorkerStartupBuilder`
- [ ] Modify `main.py` — parse `RUNNER_CAPABILITIES` env var, register capabilities

---

## 10. Verification & Tests

> Run these tests sequentially by phase. Each phase builds on the previous.

### Phase 1 — Relay + Mediator WS Connection

| # | Test | How to verify |
|---|------|---------------|
| T1 | Relay health check | `curl localhost:8080/v1/health-check` → `{"OK": true}` |
| T2 | Relay connects to Redis | Relay logs show Redis connection success on startup |
| T3 | Connections endpoint (empty) | `curl localhost:8080/health/connections` → `[]` |
| T4 | Mediator connects to relay | Start mediator with `MAIN_SERVER_URL` set → relay logs `register` event |
| T5 | Connections endpoint (1 mediator) | After T4: `GET /health/connections` → shows 1 connected mediator |
| T6 | Mediator auto-reconnect | Kill relay → restart → mediator reconnects within 1-5s, re-registers |
| T7 | Request relay: create runner | SDK `create_runner()` → relay forwards via WS → mediator creates K8s deployment |
| T8 | Mediator offline → 503 | Stop mediator → SDK request → relay returns 503 |
| T9 | GKE Gateway routing | `os.api-k.blitzy.dev/v1/relay/v1/health-check` returns 200 |
| T10 | Worker Chrome starts | Worker with `RUNNER_CAPABILITIES=["CHROME"]` → `curl localhost:9222/json/version` inside pod |
| T11 | Worker without Chrome | Worker without capability → no Chrome process, worker runs normally |

### Phase 2 — SDK + Admin Through Relay

| # | Test | How to verify |
|---|------|---------------|
| T12 | SDK sends X-Client-ID | Inspect relay logs for `X-Client-ID` header on incoming HTTP requests |
| T13 | Multi-tenant routing | Two mediators connected, each with different `client_id` → requests route to correct one |
| T14 | Secret push through relay | Admin `EnvExchangeService` → `BlitzyClient` → relay → WS → mediator receives secrets |
| T15 | Command execution e2e | SDK `submit_command()` → relay → WS → mediator → Redis → worker → result flows back |
| T16 | Runner status through relay | SDK `get_runner_status()` → relay → WS → mediator → returns status |

### Phase 3 — CDP Tunnel

| # | Test | How to verify |
|---|------|---------------|
| T17 | CDP tunnel: browser version | Local proxy `localhost:9222/json/version` → tunnels through relay → worker Chrome responds |
| T18 | CDP tunnel: screenshot | `chrome-devtools-mcp` → local proxy → relay → mediator → worker Chrome → screenshot returned |
| T19 | CDP tunnel: navigation | Navigate to URL through tunnel → page loads successfully |
| T20 | CDP tunnel: evaluation | `Runtime.evaluate` through tunnel → returns result |

### Phase 4 — Cleanup Verification

| # | Test | How to verify |
|---|------|---------------|
| T21 | CDP route code removed | No `create_chrome_route`/`delete_chrome_route` in mediator codebase |
| T22 | All flows still work | Re-run T7, T14, T15, T18 after cleanup |
| T23 | Envoy Gateway active | Verify Envoy Gateway serves its own traffic independently of relay |
| T24 | Health probes work | K8s liveness/readiness probes pass on both mediator and relay |

### Full Integration (docker-compose — Mode C)

| # | Test | How to verify |
|---|------|---------------|
| T25 | Full pipeline e2e | `docker-compose up` (Redis + relay + mediator + worker) → SDK creates runner → submits command → gets result |
| T26 | Rolling update resilience | Kill relay container → mediator reconnects → requests resume (1-2s blip) |

### Local Development Modes

- **Mode A (daily)**: `MAIN_SERVER_URL` not set → HTTP-only mode, Telepresence works as today, local Chrome for CDP
- **Mode B (piece-by-piece)**: Run component under test locally, connect to cluster relay via Telepresence or port-forward
- **Mode C (full integration)**: `docker-compose up` → full WS flow (relay + mediator + worker + Redis)

---

## 11. Worker/Runner Logging & Observability

Improve logging across the worker and runner ops for debugging and traceability.

### Current state (temporary)
- `_silence_all_loggers()` in `__main__.py` kills all BlitzyLogger handlers to prevent stdout corruption
- `processor.py` logs `command_failed` with execution_id and full stdout/stderr on failure
- `bash_session.py` logs command start/finish with execution_id

### Remaining tasks
- [ ] Fix `_silence_all_loggers()` — currently silences ALL logs including errors. Replace with JSON lines protocol so operational logs and errors are captured without corrupting stdout
- [ ] Bind `job_id` (runner_id) to `processor.py` and `bash_session.py` structlog loggers so every log line has runner context
- [ ] Long-term: Implement JSON lines protocol in `__main__.py` — output typed JSON lines (`{"type":"log",...}` vs `{"type":"result",...}`) so runner-side logs are captured without corrupting stdout. Update `session.py` `_parse_result()` to parse them
- [ ] Re-enable runner-side operational logs (scm download progress, git clone status, etc.) once JSON lines protocol is in place
- [ ] Add `runner_id` to `BLITZY_EXECUTION_ID` env var injection in `bash_session.py` wrapped command
- [ ] Verify error tracebacks from runner ops failures are fully visible in `kubectl logs` via `processor.py` `command_failed` log

---

## 12. CDP Tunnel Implementation (Option A — Room-based)

Tunnel Chrome DevTools Protocol frames between consumer CDPProxy and worker Chrome
through the relay using Socket.IO room-based routing.

> **This is capability-driven.** Tunnel rooms are only created for runners with CHROME
> capability. CDPProxy is only started by consumers that need Chrome (currently only
> `archie-job-reverse-code-generator`). Other consumers are unaffected.

### Room lifecycle

```
Runner created with CHROME capability
  → mediator emits tunnel_join {runner_id} on /tunnel
  → relay joins mediator SID to room tunnel:{runner_id}

CDPProxy connects (consumer that needs Chrome)
  → emits tunnel_open {runner_id, connection_id}
  → relay joins CDPProxy SID to room, forwards to mediator
  → mediator opens TCP to worker Chrome :9222

CDP frames flow bidirectionally via tunnel_data (room-scoped)

CDPProxy disconnects → tunnel_close → leave room
Runner deleted → mediator emits tunnel_leave → leave room → room gone
```

### Phase 1: Relay (`archie-service-relay`) — DONE

- [x] Add `on_tunnel_join(data)` to `TunnelNamespace` — joins sender SID to `tunnel:{runner_id}` room (no forwarding)
- [x] Add `on_tunnel_leave(data)` to `TunnelNamespace` — removes sender SID from `tunnel:{runner_id}` room
- [x] Add `on_tunnel_http(data)` to `TunnelNamespace` — forwards `{runner_id, request_id, method, path}` to room
- [x] Add `on_tunnel_response(data)` to `TunnelNamespace` — forwards `{runner_id, request_id, body}` to room
- [x] `/health/connections` enriched with namespace sids and tunnel room memberships
- [x] All `%s` logging converted to f-strings in tunnel.py
- [x] `SOCKETIO_NAMESPACES` constant in consts.py

### Phase 2: Mediator (`archie-client-mediator`) — DONE

- [x] Emit `tunnel_join {runner_id}` on `/tunnel` when a CHROME runner is created (`runner_service.py`)
- [x] Emit `tunnel_leave {runner_id}` on `/tunnel` when a CHROME runner is deleted
- [x] Re-emit `tunnel_join` for all active CHROME runners on reconnect (`_on_tunnel_connect`)
- [x] `TunnelRoomService` recovers rooms from DB on fresh startup (post-registration hook)
- [x] Register `tunnel_http` handler on `/tunnel` in `ws_client.py` — forwards to `TunnelForwarder.handle_http()`
- [x] `TunnelForwarder.handle_http()` — aiohttp GET to `runner-svc-{runner_id}:9222{path}`, emits `tunnel_response` back
- [x] `_connect()` separated required namespaces (/control, /tunnel) from optional (/logs). Namespace constants in consts.py
- [x] `join_tunnel_room()` checks `/tunnel` sid before emitting, logs clearly on failure
- [x] `ws_client_provider.py` — DI provider for RunnerService factories to wire tunnel callbacks
- [x] `runners.py` + `server_ready_handler.py` — pass `on_tunnel_join`/`on_tunnel_leave` to RunnerService

### Phase 3: CDPProxy (`blitzy-utils-python`) — DONE

- [x] `socketio_path` constructor param + `SERVICE_URL_RELAY` env fallback
- [x] `payload` → `data` key fix (both directions) to match mediator protocol
- [x] `start_in_background()` — dedicated thread+event loop so consumer pipeline can't starve websockets server
- [x] Debug logging in `_process_request` and `_tunnel_http_request`
- [x] `capabilities` param on `create_runner`, `RunnerCapability.CHROME`, `chrome_url` property on `RunnerSession`

### Phase 4: Worker (`archie-client-worker`) — DONE

- [x] Dockerfile: install Google Chrome
- [x] `StartChromeStep`: capability-gated via `RUNNER_CAPABILITIES=["CHROME"]`
- [x] Chrome on internal port 9333 (127.0.0.1 only — Chrome 140+ security restriction)
- [x] Python TCP proxy on 0.0.0.0:9222 → 127.0.0.1:9333 (pure stdlib, no new deps)
- [x] Readiness polling on internal port before signaling ready
- [x] Human-readable log messages throughout Chrome startup

### Phase 5: Consumer integration

- [x] `archie-job-reverse-code-generator`: `await cdp_proxy.start_in_background()`, `stop_sync()` in cleanup
- [ ] Future consumers: same pattern — check `runner_session.chrome_url`, start CDPProxy only if present
- [ ] End-to-end WebSocket frame test (chrome-devtools-mcp screenshot via tunnel)

---

## 13. Store-and-Forward & Connection Resilience

> **Canonical architecture**: `docs/STORE_AND_FORWARD.md`
> **Session logs** (in `memory/`):
> - `session_2026_04_03.md` — initial WS fixes + store-and-forward implementation
> - `session_2026_04_03_04_issues.md` — 12+ issues with root causes and fixes
> - `session_2026_04_05_13.md` — multi-worker correlation, queue endpoint move, drain simplification, race fixes

### Completed
- [x] Relay store-and-forward: Redis Streams queue + polling endpoint + drain pool
- [x] SDK `_RequestsAdapter`: `async_delivery` flag, `X-Async` header, `_poll_until_complete()`, `_request_with_retry()` with tenacity
- [x] Mediator outbound queue: `WSHttpClient` queueable param, Redis list, drain on reconnect
- [x] Response chunking: 5MB chunks for large payloads, relay reassembly
- [x] Reconnection fix: `reconnection=False` + manual `_schedule_reconnect()` with fresh `socketio.Client`
- [x] Stale sio reference fix: `WSHttpClient` now holds `ws_client` reference, `_sio` property always gets current client
- [x] Auth rejection handling: structured `SocketIOConnectionRefusedError` with status code, `ConnectionRejection` dataclass
- [x] Chunk buffer memory leak fix: cleanup `_chunk_buffers` in `on_disconnect`
- [x] Yield between chunk emits: `time.sleep(0.1)` for heartbeat processing
- [x] Ping timeout config: `PING_TIMEOUT=600`, `PING_INTERVAL=60` (configurable via env)
- [x] Worker timeout: `DEFAULT_TIMEOUT=6000` (matches archie-shared), activity timeout disabled for runner ops
- [x] Runner idempotent create: delete existing runner before recreate
- [x] Queue purge: `DELETE /api/v1/queue/{client_id}`, `purge_queue()` on BlitzyClient, called from `RunnerSession.stop()`
- [x] User-Agent header: `RELAY_USER_AGENT` constant for backend requests
- [x] Drain triggered on queue: `schedule_drain` called from relay.py when request is queued (not just on connect)
- [x] Health endpoints: `GET /api/v1/health/queue` for drain pool + per-client stats
- [x] **Multi-worker correlation** — `RedisPendingRequests` replaces in-memory `dict[str, threading.Event]`; any worker can satisfy any other worker's pending wait
- [x] **Queue endpoints under `/api/v1/queue/`** — new `queue_bp` blueprint, split from `relay.py`; reachable through GKE Gateway
- [x] **SDK poll URL uses `SERVICE_URL_RELAY`** — read once in `_RequestsAdapter.__init__`, fails fast if unset and `async_delivery=True`
- [x] **Drain simplified to XRANGE+XDEL** — removed consumer groups (XREADGROUP/XACK); no more stuck PEL entries or orphan acked entries
- [x] **Command race condition fix** — DB commit now happens before Redis enqueue; `_mark_enqueue_failure` for cleanup
- [x] **`delete_runner` best-effort** — each K8s step wrapped in try/except; soft-delete (step 7) always runs regardless of K8s failures
- [x] **Worker resource env vars** — `WORKER_CPU_REQUEST/LIMIT`, `WORKER_MEMORY_REQUEST/LIMIT`; API-payload config removed
- [x] **`RELAY_FORWARD_TIMEOUT=900s`** with matching `PENDING_TTL=960s` for large chunked responses

### Remaining
- [ ] **Client session diagnostics API** — Expose endpoint that returns connection info for a specific client: session ID (SID) on both relay and mediator side, connection uptime, last activity timestamp, namespace status (/control, /tunnel). Cross-verify relay SID matches mediator SID. Eventually enrich with: queue depth, active commands, runner count, last error.
- [ ] **Admin/utils API for manual drain trigger** — `POST /api/v1/health/drain/{client_id}` to trigger drain for specific client, `POST /api/v1/health/drain` for all clients with pending requests. Decouple drain triggering from relay blueprint.
- [ ] **BlitzyLogger `%s` format support** — Add `*args` to `info/error/warning/debug` methods so Engine.IO/Socket.IO internal logging works with `BlitzyLogger`. Currently `BlitzyLogger.info("msg %s", arg)` fails with positional args.
- [ ] **K8s `create_deployment` idempotent** — On 409 (already exists), return existing deployment instead of raising `RunnerCreationError`. Prevents race condition when SDK retries `create_runner` after relay 504 timeout.
- [ ] **`get_by_runner_id` ORDER BY** — returns first match (DEAD) instead of latest (READY) when multiple records exist. Needs `ORDER BY created_at DESC LIMIT 1`.
- [ ] **Gunicorn workers** — Increase `--workers` from 1 to 2-4 for production. Multi-worker correctness is now in place (RedisPendingRequests, Redis-backed drain state). Test with multiple workers.
- [ ] **Mediator outbound queue for non-queueable calls** — Add tenacity retry to `WSHttpClient._call()` for transient errors (`BadNamespaceError`, `ConnectionError`) on non-queueable calls. Currently non-queueable calls fail immediately on WS disconnect.
- [ ] **GKE Gateway WebSocket idle timeout** — Verify gateway's backend policy `timeoutSec` is > 660s (PING_INTERVAL + PING_TIMEOUT). If lower, WebSocket connections get killed by the gateway.

---

## 14. CDP Tunnel Resilience

> **Reference**: `docs/CDP_TUNNEL_PLAN.md` (as-built section)

### Completed
- [x] `TunnelForwarder.close_all_connections()` — closes TCP without stopping event loop, called on `/tunnel` disconnect
- [x] `WSClient._on_tunnel_disconnect` — triggers `close_all_connections()` to prevent orphaned TCP connections
- [x] CDPProxy heartbeat — background probe every 10s via existing `tunnel_http` event, no new relay events
- [x] CDPProxy circuit breaker — opens after 3 consecutive heartbeat failures, `_tunnel_http_request` fails fast, new WS rejected with 1013
- [x] CDPProxy data path fast-fail — closes all WS on circuit open, checks `_sio.connected` before each emit, prevents Socket.IO buffering stale frames
- [x] CDPProxy reconnect recovery — `_on_disconnect` opens circuit + closes WS; `_on_connect` resets circuit, clears stale futures
- [x] `is_healthy` property on CDPProxy
- [x] Relay logger migration — all 7 files migrated from stdlib `logging.getLogger` to `blitzy_utils.logger`, all `%s` logging converted to f-strings

### Remaining
- [ ] End-to-end WebSocket frame test via tunnel (chrome-devtools-mcp screenshot)

---

## 15. MCP Asset Push to Runner

> MCP tools (chrome-devtools-mcp, figma-developer-mcp) download binary files
> (screenshots, Figma assets) to the consumer job's local disk. The runner is
> a separate K8s pod. Files must be pushed to the runner for `bash mv` and
> `git add` to work.

### Completed
- [x] `write_binary_file` runner operation — base64 decode to absolute path (`blitzy-utils-python/runner_ops/operations/file_ops.py`)
- [x] `RunnerSession.push_binary_file(local_path, remote_path)` — reads binary, base64 encodes, sends to runner
- [x] `routed_push_binary_file` convenience function in `runner_ops/client.py` — no-ops locally
- [x] `push_mcp_artifact_to_runner(tool_name, args, runner_session)` in `archie-shared/mcp/utils.py` — shared function, detects Chrome screenshots vs Figma downloads
- [x] `create_chrome_mcp_config(runner_session)` in `archie-shared/mcp/utils.py` — returns (CDPProxy, chrome_config) based on runner availability
- [x] `restart_chrome(runner_session=)` in `archie-shared/mcp/utils.py` — remote via `run_bash` when session provided, local otherwise
- [x] Constants moved to `archie-shared/mcp/consts.py`: `FIGMA_MCP_DOWNLOAD_IMAGES_TOOL_NAME`, `FIGMA_ASSETS_DIR`, `REMOTE_CHROME_CONFIG`
- [x] `capabilities.py` renamed to `consts.py` in `blitzy-utils-python/runner_ops/` — all imports updated
- [x] `archie-job-reverse-code-generator/helper.py` simplified — uses shared functions from `archie-shared/mcp/utils.py`

---

## 16. Worker RQ Migration (SimpleWorker)

> Replaced hand-rolled BLPOP loop with RQ's SimpleWorker for proper job
> lifecycle management. Jobs are now tracked through Redis registries
> instead of being destructively consumed.

### Completed
- [x] `src/blitzy_worker/tasks.py` — RQ task function `execute_command(**kwargs)`, module-level worker reference
- [x] `BlitzyWorker.run()` replaced — `SimpleWorker(queues=[queue]).work()` instead of BLPOP loop
- [x] `_rq_exception_handler` — publishes FAILED `ExecutionResult` when job raises, so mediator/SDK doesn't hang
- [x] `WORKER_TASK_FUNC` constant in mediator `consts.py` — importable path of the worker task function
- [x] `command_service.py` updated to use `WORKER_TASK_FUNC` constant
- [x] Queue name fix — use `command_queue_base` (bare name) for RQ's `Queue()`, not `command_queue_name` (which has `rq:queue:` prefix)
- [x] Circular import resolved — `tasks.py` no longer imports from `worker.py`
- [x] Worker hardening: Chrome `stdout/stderr=DEVNULL`, `self._process` stored, TCP proxy bind error handling on main thread, `RUNNER_CAPABILITIES` JSON parsing guarded, `EVENT_DATA` key access validated

### Job lifecycle (new)
```
Queue → StartedJobRegistry → FinishedJobRegistry (success)
                           → FailedJobRegistry (exception)
                           → stays in StartedJobRegistry (crash → requeued on next worker startup)
```

### Remaining
- [ ] **`_parse_result` stderr bug** — when worker status is FAILED, `_parse_result` reads `stderr` (empty) but the actual error is in `stdout` as JSON. Fix: try parsing stdout for structured error before falling back to stderr.
- [ ] **Redis Streams migration** — current RQ approach covers 95% of failure scenarios. The 5% gap (pod can't restart at all) would need true ACK-based semantics (Redis Streams with XACK/XCLAIM). Tracked as future consideration.
