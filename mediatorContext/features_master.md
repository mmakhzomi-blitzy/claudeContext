# Master Feature Inventory — Mediator + Relay

**Scope**: every capability the current Python implementation of `archie-client-mediator` (customer-side) and `archie-service-relay` (platform-side) provides. Neutral enumeration — no recommendations.

**Status legend**:
- **MUST**: required in rewrite. Removing/breaking is a regression.
- **SHOULD**: required for current customer use cases; could be deferred to a later milestone if scoped explicitly.
- **COULD**: optional. Convenient but not blocking.

References to deeper docs use bare filenames (no paths) so this set is portable.

---

## A. SDK-facing HTTP API (mediator)

Customer SDK (`BlitzyClient` in `blitzy-utils-python`) talks to mediator over HTTP REST. Full endpoint contract: see `contract_http_api.md`.

| # | Feature | What it does | Why it exists | Status |
|---|---|---|---|---|
| A1 | Runner lifecycle CRUD | `POST /api/v1/runners`, `GET /api/v1/runners/{job_id}`, `DELETE /api/v1/runners/{job_id}` — create/read/delete K8s worker pods (and per-job Redis queues) | Consumer jobs need ephemeral worker pods to execute bash commands remotely | MUST |
| A2 | Command submission | `POST /api/v1/jobs/{job_id}/commands` — enqueue a bash command for a runner | Replaces in-process `BashSession`; lets consumer jobs run commands in worker pods | MUST |
| A3 | Command status polling | `GET /api/v1/commands/{execution_id}/status` — return current state + stdout/stderr | Consumer SDK polls until terminal status | MUST |
| A4 | Restart bash session | `POST /api/v1/jobs/{job_id}/restart-session` — sends control command to worker | Recovery for shells that get stuck or pollute env | MUST |
| A5 | Command archive query | `GET /api/v1/command-archive` — paginated query of historical command executions | Audit and post-hoc debugging by SDK consumers | SHOULD |
| A6 | Vault secret CRUD | `POST/GET/DELETE /api/v1/secrets[/...]` — store/list/read/delete customer secrets at `environment/{env_id}` Vault paths | Customer-managed env secrets exposed to runner pods | MUST |
| A7 | SCM repo info | `GET /api/v1/scm/{git_project_repo_id}/info` — fetch org, installation_id, type | Worker needs SCM metadata to clone | MUST |
| A8 | SCM access token | `GET /api/v1/scm/{git_project_repo_id}/access_token` — short-lived clone token | Worker uses to authenticate `git clone` | MUST |
| A9 | Registration | `POST /api/v1/register` — register mediator with main backend | Bootstrap; binds mediator identity to a customer | MUST |
| A10 | Health probes | `GET /api/v1/health-check`, `/api/v1/uptime-check`, `/api/v1/redis-check` | K8s liveness/readiness + LB checks | MUST |
| A11 | OTLP log receiver | `POST /otlp/v1/logs` — accept OTLP JSON log batches from in-cluster OTel collector | Customer-side log fan-in point; mediator forwards to relay over `/logs` | SHOULD |

## B. SDK-facing HTTP API (relay)

Relay exposes a proxy surface — SDK can talk to relay instead of mediator directly, then relay forwards to mediator over WS.

| # | Feature | What it does | Why it exists | Status |
|---|---|---|---|---|
| B1 | Request relay (sync) | `*  /<path:path>` — proxy SDK HTTP request to online mediator's `/control` WS | One stable URL for SDK; mediator can move/restart behind it | MUST |
| B2 | Request relay (async + queue on miss) | Same path + `X-Async: true` header — queue the request when mediator offline, return 202 with `X-Relay-Request-ID` | SDK survives mediator restarts without dropping requests | MUST |
| B3 | Queue request poll | `GET /api/v1/queue/requests/{request_id}` — poll for queued request completion | Async path's terminal status retrieval | MUST |
| B4 | Queue purge | `DELETE /api/v1/queue/{client_id}` — drop all queued requests for a client | Manual recovery / cleanup | COULD |
| B5 | Connection health introspection | `GET /api/v1/health/connections` — list all connected mediators with namespace/room state | Vendor-side operational visibility | SHOULD |
| B6 | Queue / drain health | `GET /api/v1/health/queue` — drain pool stats + per-client queue depth + age | Vendor-side operational visibility | SHOULD |
| B7 | Basic health probes | `GET /api/v1/health-check`, `/api/v1/uptime-check` | K8s probes | MUST |

## C. Customer mediator → relay transport (Socket.IO today; replaceable)

Mediator opens a persistent connection to relay across three logical namespaces. Full event/payload catalog: see `contract_ws_protocol.md`.

| # | Feature | What it does | Why it exists | Status |
|---|---|---|---|---|
| C1 | `/control` namespace | Auth + relay→mediator HTTP-style request dispatch + mediator→relay response | Carries the SDK request stream over a single persistent egress connection | MUST |
| C2 | `/tunnel` namespace | Room-based forwarding of CDP frames + HTTP discovery between consumer's CDPProxy and mediator's TunnelForwarder | Chrome DevTools Protocol passthrough for browser-control runners | MUST |
| C3 | `/logs` namespace | Forward OTLP log batches from mediator (one connection per customer) to vendor's OTel forwarder | Customer telemetry egress without customer cluster having direct vendor-SaaS access | SHOULD |
| C4 | Authentication on connect | Mediator presents `{api_key, url}`; relay validates against admin service; returns `client_id` | One-step auth; binds the WS session to a customer identity | MUST |
| C5 | Persistent connection per mediator pod | One mediator pod = one WS connection per namespace to relay | Single egress chokepoint; customer cluster firewalls only need to allow outbound TLS to relay | MUST |
| C6 | Manual reconnect with backoff | Mediator side: detect disconnect → discard old socket.io.Client → new client → 1s/1.5s/...5s capped retry → re-register | python-socketio's built-in reconnect has a known thread-mode bug; this is the workaround | MUST |
| C7 | Tunnel room recovery after reconnect | After re-registration, mediator re-emits `tunnel_join` for every active runner | New WS session loses room membership; must rejoin to receive CDP frames | MUST |
| C8 | Auth failure = hard exit | 401 from relay → mediator `SystemExit` → pod CrashLoopBackOff | Wrong API key shouldn't tight-loop reconnect forever | MUST |
| C9 | Response chunking (≥6 MB) | Large mediator→relay responses split into `response_chunk` events with 0.1s sleep between | Socket.IO single-frame size limits + ping/pong scheduling | SHOULD |
| C10 | Backend-request via WS | Mediator-initiated outbound HTTP relayed through `/control` via `backend_request` event | Mediator can call backend services without customer-cluster direct egress | MUST |
| C11 | Outbound store-and-forward | When relay is offline, `WSHttpClient.post(queueable=True)` writes to `mediator:outbound_queue` Redis list; drained on reconnect | Webhook forwarding & similar fire-and-forget must survive relay restarts | MUST |
| C12 | Synthetic 202 response on queue | When `queueable=True` and call queued, return `WSResponse(status=202, body={"queued": True})` | Callers don't need to know about queueing | MUST |

## D. Inbound store-and-forward (SDK ↔ relay when mediator offline)

When mediator is offline at the moment the SDK calls relay, relay queues the request for later delivery. See `STORE_AND_FORWARD.md` for as-built details.

| # | Feature | What it does | Why it exists | Status |
|---|---|---|---|---|
| D1 | Inbound queue per client | Redis Stream `relay:requests:{client_id}` — capped at `REQUEST_QUEUE_MAXLEN` (1000) | Survive mediator downtime up to 1000 backed-up requests per customer | MUST |
| D2 | Pending response cache | Redis String `relay:pending:{request_id}` JSON, TTL 960s | Status polling target for SDK; lifetime exceeds the 900s `RELAY_FORWARD_TIMEOUT` | MUST |
| D3 | Drain pool | Background pool of `DRAIN_WORKER_COUNT` (default 4) threads that pop client IDs from `relay:drain:pending` Set, lock per-client, drain up to `DRAIN_BATCH_SIZE` (100) per turn | Bounded concurrent drain across multiple connected customers | MUST |
| D4 | Distributed drain lock | Redis `SETNX relay:drain:lock:{client_id}` 60s TTL — prevents two relay workers racing on the same client | Multi-worker gunicorn pool needs coordination | MUST |
| D5 | Multi-worker correlation | `RedisPendingRequests` — gevent-cooperative 50ms-poll on Redis key, so any gunicorn worker can satisfy any other worker's pending wait | Multi-worker gunicorn means request-emitter and response-receiver are often different processes | MUST |
| D6 | `XRANGE+XDEL` drain pattern | Read oldest entry, emit, await response, then `XDEL` only on success — failures stay in stream and retry naturally | Cleaner than consumer-group semantics; avoids accumulated PEL entries we hit in practice | MUST |
| D7 | Header contract for queueing | `X-Async: true` opts in; `X-Relay-Request-ID` returned to distinguish queued 202s from mediator-originated 202s | SDK uses header presence to decide whether to enter poll loop | MUST |

## E. CDP tunnel (Chrome DevTools Protocol passthrough)

Customer's consumer job runs `chrome-devtools-mcp` which speaks CDP over a localhost:9222 server. CDPProxy (in consumer process) tunnels that traffic via relay → mediator → runner pod's Chrome. See `CDP_TUNNEL_PLAN.md`.

| # | Feature | What it does | Why it exists | Status |
|---|---|---|---|---|
| E1 | Room-based tunnel routing | Both CDPProxy (consumer) and TunnelForwarder (mediator) join `tunnel:{runner_id}` Socket.IO room; events forward room-wide excluding sender | Multi-tenant tunnel multiplexing on one relay namespace | MUST |
| E2 | TCP forwarding (binary CDP frames) | `tunnel_open` → mediator opens TCP to worker pod; `tunnel_data` → bidirectional bytes; `tunnel_close` → tear down | The actual CDP protocol is binary WS frames; we transport bytes | MUST |
| E3 | HTTP discovery proxy | `tunnel_http` / `tunnel_response` carry CDP HTTP discovery (`/json/version`, `/json/list`) | chrome-devtools-mcp uses these endpoints to find browser targets | MUST |
| E4 | Heartbeat + circuit breaker (CDPProxy) | CDPProxy polls `/json/version` every 10s; after 3 fails, opens circuit → fast-fail incoming WS connections | Prevents 30s hangs when relay is down mid-session | MUST |
| E5 | Auto-recovery on reconnect | Circuit closes when probe succeeds; chrome-devtools-mcp retries and tunnel rebuilds | Continuous browser-control work survives relay restarts | MUST |
| E6 | MCP asset push to runner | When MCP tool downloads files to consumer disk (screenshots, Figma assets), `push_binary_file` runner op transfers to runner | `bash mv` and `git add` on the runner expect the files locally | SHOULD |

## F. Runner / worker pod lifecycle (mediator side)

Mediator creates K8s Deployments + Services for worker pods at runtime via the K8s Python client. See `ARCHITECTURE_REFERENCE.md`.

| # | Feature | What it does | Why it exists | Status |
|---|---|---|---|---|
| F1 | Worker Deployment creation | `kubernetes_service.create_deployment()` builds `V1Deployment` with image, resources, env, image-pull secrets | One runner = one Deployment (1 pod replica) | MUST |
| F2 | Runner Service + HTTPRoute (CHROME) | For Chrome runners, mediator also creates a ClusterIP Service + Gateway-API HTTPRoute targeting port 9222 | CDP tunnel needs an addressable host:port | MUST |
| F3 | Env-var injection into worker | `WORKER_*` vars, Redis creds, `JOB_ID`, queue names, `CORRELATION_ID`, OTel service-name annotation, custom env from config | Worker needs Redis + identity + observability context | MUST |
| F4 | Worker affinity / nodeSelector / tolerations | `WORKER_AFFINITY`/`WORKER_NODE_SELECTOR`/`WORKER_TOLERATIONS` env vars (JSON), deserialized to K8s SDK types and applied to V1PodSpec | Customer can pin workers to specific node pools | SHOULD |
| F5 | Runner readiness signal | Worker publishes `server_ready` event to `blitzy-results` queue; mediator handler marks `JobServer.status = READY` | SDK waits for READY before submitting commands | MUST |
| F6 | Image pull secrets | `IMAGE_PULL_SECRET_NAME` env injected as `V1LocalObjectReference` on PodSpec | Customer's private registry support | MUST |
| F7 | Runner teardown | `DELETE /api/v1/runners/{job_id}` → archive commands, delete Deployment, delete Service+HTTPRoute (if CHROME), mark JobServer deleted, drop queue | Ephemeral resources; long-lived clusters need cleanup | MUST |
| F8 | Command archival on teardown | `command_executions` rows server-side-bulk-inserted into `command_execution_archive`, then deleted from live table | Live table stays small; history preserved | MUST |

## G. Worker dispatch (Redis queues)

| # | Feature | What it does | Why it exists | Status |
|---|---|---|---|---|
| G1 | Per-runner job queue | RQ queue `blitzy-worker-queue-{runner_id}` — mediator enqueues `execute_command` jobs; worker pod consumes | Routes commands to the right runner | MUST |
| G2 | Shared results queue | RQ queue `blitzy-results` — workers publish `command_result` and `server_ready` events; mediator's in-process RQ worker subprocess consumes | One return path for all runners back to mediator | MUST |
| G3 | RQ worker subprocess | `multiprocessing.Process` running RQ `Worker.work()` in mediator container | Process the results queue without blocking gunicorn | MUST |
| G4 | Watchdog parent-kill | Daemon thread polls `worker_process.is_alive()` every 5s; SIGTERMs parent on death | Make CrashLoopBackOff observable; K8s restarts pod | MUST |
| G5 | Result handler updates DB | `process_job` routes `command_result` → updates `CommandExecution` row (status, exit_code, stdout, stderr, timestamps) | DB is SDK's polling source-of-truth | MUST |

## H. SCM token flow

Currently mediator forwards SCM token requests directly to backend (temporary; mediator integration with Vault for SCM is planned later). See `project_scm_integration.md`.

| # | Feature | What it does | Why it exists | Status |
|---|---|---|---|---|
| H1 | Direct backend token fetch | `GET /api/v1/scm/{repo_id}/access_token` → mediator → backend (via `WSHttpClient`) → returns short-lived token | Worker needs SCM creds to clone | MUST |
| H2 | Repo metadata fetch | `/info` endpoint similarly forwards to backend | Worker needs to know how to authenticate (app vs installation) | MUST |
| H3 | GHES webhook receiver | `POST /api/v1/ghes/webhook/{hostname}/{app_id}` — verify HMAC, forward to backend (queueable) | GHES webhook event ingestion | MUST |
| H4 | GHES proxy | `POST /api/v1/ghes/proxy` — internal WS-dispatched, proxies any GHES API call with chosen auth strategy | Workers and orchestrators reach GHES through mediator | MUST |

## I. Vault integration

| # | Feature | What it does | Why it exists | Status |
|---|---|---|---|---|
| I1 | Customer secret CRUD | Vault paths under `environment/{env_id}` | Per-environment customer secrets | MUST |
| I2 | App auth via K8s SA | Mediator authenticates to Vault using its K8s service account JWT | No static Vault token; cluster identity bound | MUST |
| I3 | Secret cache in mediator | `SecretsStore` — in-process cache with explicit invalidation | Avoid hammering Vault on every request | SHOULD |
| I4 | Worker secret injection | Worker reads its own secrets from Vault using `WORKER_SERVICE_ACCOUNT` | Secrets don't transit through mediator → worker as env vars | MUST |

## J. Observability

Today mostly logs; metrics/traces planned. See `OBSERVABILITY_PLAN.md`.

| # | Feature | What it does | Why it exists | Status |
|---|---|---|---|---|
| J1 | Structured JSON logs | `blitzy_utils.logger` with global fields `image_tag`, `client_id`, `correlation_id` | Multi-tenant log triage | MUST |
| J2 | Per-request correlation ID | Middleware extracts `X-Correlation-ID`, propagates to log fields and worker env | Trace one request end-to-end across mediator → worker | MUST |
| J3 | OTel collector → mediator `/otlp` → relay `/logs` → forwarder | Logs + (future) metrics + traces flow through this chain — never directly from customer cluster to vendor SaaS | Customer security: no direct vendor-SaaS egress | MUST |
| J4 | Relay deep health endpoints | `/health/connections` and `/health/queue` already expose registry state + drain stats | Operational visibility without extra metric pipeline | MUST |
| J5 | Watchdog logs cause-of-death | Logs exit code before SIGTERMing parent | Diagnosability of RQ subprocess crashes | SHOULD |

## K. Configuration & deployment

| # | Feature | What it does | Why it exists | Status |
|---|---|---|---|---|
| K1 | Env-driven config via ConfigMap | `blitzy-config` ConfigMap mounted via `envFrom` | Single source of truth for non-secret config | MUST |
| K2 | Secrets via Vault (preferred) or K8s Secret | Postgres credentials, Redis password, API key | Standard K8s secret handling | MUST |
| K3 | CSI volume mount for AKV / external secret stores | `clientMediator.extraVolumes` + `extraVolumeMounts` (Helm chart) | Customer can mount Azure KV / similar at `/mnt/secrets-store` | SHOULD |
| K4 | Pod scheduling overrides | `clientMediator.scheduling.{affinity,nodeSelector,tolerations}` (chart) + per-runner `worker.scheduling.*` (forwarded to mediator via configmap → mediator → V1PodSpec) | Customer pins blitzy workloads to specific node pools | SHOULD |
| K5 | Multi-arch container images | linux/amd64 + linux/arm64 | Customer node-pool architecture variability | SHOULD |
| K6 | Mirror via customer's registry | All upstream images mirrored to `infra-images` or customer's private registry | Air-gapped / approved-image-registry customers | MUST |

## L. Concurrency model (current)

| # | Feature | What it does | Why it exists | Status |
|---|---|---|---|---|
| L1 | Mediator: 1 gunicorn worker × 32 threads | Single Python process, in-process state (WSClient, TunnelForwarder asyncio loop) | Simpler state management; avoids cross-worker coordination | MUST (intent — rewrite may change) |
| L2 | Mediator: 32-slot ThreadPoolExecutor for `/control` request dispatch | Off Engine.IO receive thread to avoid blocking ping/pong | Prevent connection drop under load | MUST |
| L3 | Mediator: TunnelForwarder asyncio loop in dedicated daemon thread | Bridges sync Socket.IO callbacks with async TCP `open_connection` | Chrome traffic = many concurrent TCP streams | MUST |
| L4 | Mediator: outbound drain daemon thread | Drains `mediator:outbound_queue` on every `/control` reconnect | Don't block Engine.IO receive thread | MUST |
| L5 | Relay: 10 gunicorn workers × gevent class | gevent-monkey-patched cooperative multitasking | Single Python process handles many concurrent WS connections | MUST (intent — rewrite may change) |
| L6 | Relay: 4 drain-pool worker threads | Picks up `relay:drain:pending` Set entries, drains per-client | Bounded concurrent drain across customers | MUST |

## M. Hard constraints / known invariants

Things that must remain true regardless of framework choice.

| # | Constraint | Reason |
|---|---|---|
| M1 | Single customer-egress chokepoint | Customer security teams reject anything that makes direct outbound calls to vendor SaaS |
| M2 | HTTP/2 (or HTTP/1.1) over TLS — no special protocols | Customer corporate proxies may not pass arbitrary L4 traffic |
| M3 | Relay knows `client_id` per connection | Multi-tenant routing + telemetry tagging depend on it |
| M4 | Request body up to 100 MB | Some SDK calls carry large payloads (code generation outputs, etc.) |
| M5 | Request timeout up to 15 min (`RELAY_FORWARD_TIMEOUT=900s`) | Some commands take long to complete; SDK polls for terminal status |
| M6 | Outbound queue must persist across mediator restart | Otherwise a redeploy drops in-flight queueable requests |
| M7 | RQ queue (Redis) is the only worker-dispatch channel | If we change worker dispatch, the `archie-client-worker` repo also changes |
| M8 | Customer's K8s namespace is the security perimeter | Cross-namespace access is not permitted; runner pods stay in mediator's namespace |

## N. Things that are NOT features (explicit non-features)

For clarity in the rewrite — these aren't required:

| # | Non-feature | Notes |
|---|---|---|
| N1 | Per-customer Redis isolation in shared relay | Today one Redis serves all customers; keys are namespaced by `client_id`. No tenant DB isolation. |
| N2 | Customer SSO / OAuth | Auth is exclusively API key |
| N3 | Encryption-at-rest for queued requests | Outside scope; relies on Redis-level/cloud-provider encryption |
| N4 | Per-customer rate limiting | No rate limits enforced anywhere today |
| N5 | gRPC anywhere | Considered and rejected (CDP tunnel doesn't fit) |
| N6 | Persistent mediator-side request history | Live `command_executions` rows + archive table are the only history; no immutable audit log |

---

## Open questions captured during inventory (covered in `requirements_rewrite.md`)

- Multi-mediator architecture: how does scale-out work?
- Reconnect semantics: how much state is recoverable across a relay restart?
- Performance target: 250 → 1000–1500 concurrent runners — where exactly is the bottleneck?
- WS connection-count trade: one beefy connection vs. N connections per customer?

Cross-references:
- HTTP surface details → `contract_http_api.md`
- Redis key catalog → `contract_redis_keys.md`
- Current WS protocol → `contract_ws_protocol.md`
- SDK contract & allowed minimal changes → `contract_sdk.md`
- Rewrite capability spec → `requirements_rewrite.md`
- Framework comparison → `framework_evaluation.md`
