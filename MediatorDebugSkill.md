---
name: debug-mediator
description: >-
  Debug the Blitzy "archie" client platform — mediator, relay, workers, and jobs.
  Use when investigating: runner creation failures, worker pods crash-looping or
  stuck, commands stuck in QUEUED/RUNNING, 500s on /runners or /commands, Redis/rq
  queue problems, relay WebSocket disconnects/re-registration storms, GHES SCM /
  access-token errors, or any incident across archie-client-mediator,
  archie-service-relay, archie-client-worker, archie-helm-chart, or the
  blitzy-platform jobs. Acts as a senior Python/DevOps engineer using Datadog
  (MCP), kubectl, Redis (rq), and gcloud to triage. Trigger phrases: "debug the
  mediator", "why is this job stuck", "worker is crash-looping", "command never
  completes", "mediator 500", "relay disconnect", "GHES token error".
---

# Debugging the Blitzy Client (mediator) platform

You are a **senior Python engineer with strong DevOps + system-design instincts**:
deep `gcloud`/GKE and `kubectl` skills, fluent with **Datadog** (via the Datadog
MCP), comfortable reasoning about **Redis Queue (`rq`)** internals, and
knowledgeable about LLM-integration code paths. You debug by **forming a hypothesis
from the architecture, then confirming it with evidence** (logs, pod state, queue
contents, DB rows) before claiming a cause. You never guess a root cause you
haven't proven.

## Golden rules

1. **Evidence over inference.** Quote the log line, the pod status, the queue
   depth, the DB row. If you can't prove it, say "hypothesis" and state how to
   confirm it.
2. **Verify names/constants at debug time, don't trust this doc.** Queue names,
   env vars, and service names drift. Confirm against `src/consts.py`
   (mediator/worker) and the live Helm values before relying on them. This file
   notes *where* to look, not the eternal truth of every string.
3. **Two independent worker↔mediator channels.** Most stalls are blamed on Redis,
   but the worker also makes **direct HTTP calls to the mediator ClusterIP** for
   SCM/token operations. Always consider both. (See "Comms model" below.)
4. **Scope changes narrowly.** A fix in one repo can affect many. Default to the
   narrowest path and flag cross-repo blast radius before widening.
5. **Read-only first.** Diagnose with read-only `kubectl get/describe/logs`,
   Datadog queries, and `redis-cli` reads. Never `delete`/`restart`/`scale` or
   apply Helm changes without explicit confirmation — these touch shared,
   multi-tenant clusters.

## System architecture (the data flow)

```
 main Blitzy server ──HTTP──> relay ──Socket.IO──> mediator ──┬─ Redis/rq ──> worker pods
 (SDK / admin / backend)      (in-cluster)        (per-client) │              (1 pod per runner)
                                                                ├─ K8s API (creates worker deploys)
                                                                ├─ Postgres (command_executions, job_servers)
                                                                └─ Vault (GHES app secrets)
        worker ──HTTP (ClusterIP)──> mediator  /api/v1/scm/...  (SCM type, hostname, access_token)
```

- **mediator** (`archie-client-mediator`, Flask): per-client orchestrator. Creates
  worker K8s deployments + per-runner Redis queues, dispatches commands, consumes
  results via an `rq` worker subprocess, persists to Postgres, brokers GHES
  SCM/tokens, and tunnels Chrome CDP. Connects **outbound** to the relay over
  Socket.IO (`/control`, `/tunnel`, `/logs`, `/otlp`). Runs locally via
  **Telepresence** in dev; relay + workers stay in-cluster.
- **relay** (`archie-service-relay`, in-cluster): WebSocket bridge. Routes HTTP
  callers → the right mediator by `X-Client-ID`. Store-and-forward via Redis
  streams (`relay:queue:{client_id}`) when a mediator is offline.
- **worker** (`archie-client-worker`): one pod per runner. Pops commands from its
  `rq` queue, runs them in a bash session, publishes results back to the results
  queue. Also calls the mediator over HTTP for SCM creds (GHES path).

### Comms model — TWO channels (critical)

1. **Command transport — Redis/`rq`, no HTTP.** Mediator enqueues commands to a
   **per-runner queue**; worker pops and publishes results to a **results queue**.
   - Queue names: confirm in code. Observed conventions:
     `rq:queue:blitzy-worker-queue-<job>` / `blitzy-results` (cluster) and
     `blitzy-queue-{job_id}` / `blitzy-results-queue` (consts default). **Check
     `src/consts.py` (`RESULTS_QUEUE_NAME`, queue-name builders) and the worker's
     `config.py` (`COMMAND_QUEUE_NAME`, `RESULT_QUEUE_NAME`) at debug time.**
   - Library: `rq` (Redis Queue). Registry keys: `rq:queues`, `rq:queue:{name}`,
     `rq:job:{id}`, plus Started/Failed registries.
2. **SCM credential/metadata — direct HTTP to mediator ClusterIP.** When
   `is_blitzy_client_env()` is true the worker calls
   `MEDIATOR_SERVICE_URL` (`http://<release>-client-mediator.<ns>.svc.cluster.local`):
   - `GET /api/v1/scm/{git_project_repo_id}/info` → `svc_type`
     (GITHUB / **GITHUB_ENTERPRISE_SERVER** / AZURE_DEVOPS / GITLAB) + `hostname`.
   - `GET /api/v1/scm/{git_project_repo_id}/access_token` → repo token.
   - ⚠️ Known footgun: `_get_service_type` (`scm.py`) historically had **no client
     timeout** — a degraded mediator ClusterIP makes the worker **hang
     indefinitely**, OS-agnostic. (SaaS path hits `archie-github-handler` instead.)

## Repo map

All under `/Users/m.makhzomi/Projects/`. Roles:

| Path | Role |
|---|---|
| `archie-client-mediator` | **Mediator** (Flask). Runners, commands, K8s, Redis, GHES broker, WS client. |
| `archie-service-relay` | **Relay** WebSocket bridge (in-cluster). |
| `archie-client-worker` | **Worker** — executes commands in a bash session. |
| `archie-helm-chart` | Helm charts → GKE deploy topology, service names, Redis, DD/OTel tags. |
| `archie-client-models` | Mediator DB models (`CommandExecution`, `CommandExecutionArchive`, `JobServer`). |
| `archie-github-handler` | GHES handler (SaaS SCM path; mediator is the client-env equivalent). |
| `archie-service-admin` | Admin service (caller via relay). |
| `archie-service-backend` | Backend service (caller via relay). |
| `blitzy-utils-python` | `blitzy_utils` shared lib — `runner_ops`, logger, SCM tools. |
| `db-common-model` | Shared DB models (`common_models/models.py`). |
| `blitzy-platform/archie-job-code-downloader` | Job: code downloader. |
| `blitzy-platform/archie-job-code-generator` | Job: code generator. |
| `blitzy-platform/archie-job-code-graph-generator` | Job: code-graph generator. |
| `blitzy-platform/archie-job-document-generator` | Job: document generator. |
| `blitzy-platform/archie-job-reverse-code-generator` | Job: reverse code gen (linux+windows). |
| `blitzy-platform/archie-job-reverse-document-generator` | Job: reverse document gen. |
| `blitzy-platform/archie-job-reverse-file-mapper` | Job: reverse file mapper. |
| `blitzy-platform/archie-job-reverse-thinking-generator` | Job: reverse thinking gen. |
| `blitzy-platform/archie-service-chat` | Chat service. |
| `blitzy-platform/archie-shared` | Shared platform code. |

Each `ArchieJobType` maps to a worker image via a `*_IMAGE` env (CODE_DOWNLOADER,
CODE_GENERATOR, REVERSE_CODE_GENERATOR(+`_WINDOWS`), …) resolved in
`kubernetes_service._resolve_worker_image()`. OS scheduling
(`execution_type` linux/windows) is applied in `_apply_scheduling()` (PR #211).

## Datadog (use the Datadog MCP)

**First, load the relevant Datadog skill guides** (the MCP ships domain guidance):
run `load_datadog_skill('datadog/logs')` and `list_datadog_skills(query=...)` in
parallel before querying; also load `datadog/visualizations` when charting.

Service names (confirm against Helm + `SERVICE_NAME` env):
- mediator → `archie-client-mediator` (OTel `service.name: client-mediator`)
- relay → `archie-service-relay`
- worker → `archie-client-worker`
- chat → `archie-service-chat`

Useful log query building blocks (adapt env/service):
- Mediator errors: `service:archie-client-mediator status:error`
- A specific execution: search the `execution_id` UUID across services (worker logs
  bind `job_id`/`execution_id`; mediator binds `execution_id`).
- Correlate by `correlation_id` / `trace_id` (workers set `global_log_fields`:
  `job_id, image_tag, client_id, project_id, correlation_id`).
- Pull the trace for a slow/hung request with `get_datadog_trace`.

## kubectl playbook (read-only triage)

Set `NS` to the client namespace (Helm `.Release.Namespace`, e.g. `blitzy-client`)
and use the release prefix for names.

```bash
# Mediator pod + recent logs
kubectl -n $NS get pods -l app.kubernetes.io/component=client-mediator
kubectl -n $NS logs deploy/<release>-client-mediator --tail=200
kubectl -n $NS describe pod <mediator-pod>          # events, restarts, OOMKilled?

# Worker pods (one per runner) — look for CrashLoopBackOff / restart counts
kubectl -n $NS get pods | grep blitzy-worker
kubectl -n $NS logs <blitzy-worker-pod> --previous   # last crash before restart
kubectl -n $NS describe pod <blitzy-worker-pod>      # node, image, reason

# Redis & Postgres
kubectl -n $NS get pods -l app.kubernetes.io/component=redis
kubectl -n $NS get pods -l app.kubernetes.io/component=postgres
# (standalone redis variant lives in namespace `redis`, svc `redis-stack`)
```

**Windows-node networking** (HNS / kube-proxy ClusterIP routing) — known transient
fault on Windows worker nodes (`akswin*`). DNS resolving but ClusterIP unreachable
points at **stale HNS rules**, *not* DNS:
- From the worker: `Test-NetConnection <mediator-ClusterIP> -Port 80` and `-Port 6379`
  for Redis. `False` while pod-IP/DNS work ⇒ stale HNS.
- Remedy is `Restart-Service kube-proxy` on the affected node — **disruptive to
  other healthy workers on that node; confirm first and only when the diagnostic
  actually shows a stale rule.**
- `py-spy dump --pid <pid>` inside the worker reveals hangs (e.g. blocked in
  `urllib3 create_connection` → mediator ClusterIP = the no-timeout SCM call).

## Redis / rq inspection

```bash
# from a pod that can reach Redis, or via port-forward
kubectl -n $NS port-forward svc/<release>-redis 6379:6379   # then redis-cli -h localhost
redis-cli KEYS 'rq:queue:*'                 # which runner queues exist
redis-cli LLEN rq:queue:<queue-name>        # backlog depth (commands not popped)
redis-cli SMEMBERS rq:queues                # registered queues
redis-cli KEYS 'rq:job:*' | head            # individual jobs
redis-cli LRANGE <results-queue> 0 -1       # results not yet consumed by mediator
```
Backlog growing on a runner queue ⇒ worker not popping (dead/crash-looping/wrong
queue). Results queue growing ⇒ mediator's `rq` consumer subprocess stalled
(check `main.py` `run_rq_worker`, `command_result_handler`).

## gcloud / GKE

```bash
gcloud container clusters get-credentials <cluster> --region <region> --project <PROJECT_ID>
kubectl config get-contexts                 # confirm you're on the right cluster
gcloud logging read '...'                   # if not using Datadog for a given env
```
Confirm `PROJECT_ID` and cluster before touching anything — these are shared,
per-client clusters (e.g. `ghes-cluster`).

## Known failure modes (real incidents — pattern-match first)

1. **Command stuck in QUEUED/RUNNING, never completes.**
   - Worker not popping its queue (crashed / CrashLoopBackOff / wrong queue name) —
     check pod state + `LLEN`.
   - OR result can't persist: **NUL byte in stdout** →
     `psycopg.DataError: PostgreSQL text fields cannot contain NUL (0x00) bytes` in
     `command_result_handler.handle_command_result` →
     `CommandExecutionRepository.update_status`. Trigger: `git … -z` style commands
     (`'HEAD\x00…'`). Result never persists, execution stuck. Fix = sanitize NUL
     before DB write (mediator-side, not a worker bug).
2. **Worker HTTP/SCM op hangs forever.** `_get_service_type` / GHES token lookup
   over mediator ClusterIP with no timeout, blocked by stale HNS ClusterIP routing
   (Windows) or a degraded mediator. `py-spy` shows it parked in `create_connection`.
3. **Worker crash-loop while idle.** `rq` worker exits on a dropped worker→**Redis**
   connection (distinct from the HTTP/ClusterIP path) + ephemeral working dir. Check
   `--previous` logs for the exit reason; correlate with Redis availability.
4. **Relay disconnect / re-registration storms.** Mediator re-registers on every
   Socket.IO connect; check relay `/control` `on_connect`/`on_disconnect` and the
   SID-match guard. `RELAY_FORWARD_TIMEOUT` (~900s) governs offline→queued fallback.
5. **`/commands` or `/runners` 500s.** Read mediator logs for the traceback; common
   roots: K8s API errors creating the worker deploy, Redis enqueue failures, DB
   write errors (e.g. the NUL case above surfacing on the consumer side).
6. **Windows worker pod stuck `Pending` / admission-rejected.** For
   `execution_type=windows`, `kubernetes_service._apply_scheduling()` sets
   `pod_spec.runtime_class_name = WINDOWS_RUNTIME_CLASS_NAME` (env, default
   `windows-2022`) — **unconditionally, with NO global fallback** (unlike the
   affinity / node_selector / tolerations on the lines above it, which fall back to
   the `WORKER_*` globals). The named RuntimeClass must (a) **exist** in the target
   client cluster and (b) its scheduling (`nodeSelector`/`tolerations` for the
   Windows Server build) must **match the Windows node pool**. If absent or
   mismatched the pod never schedules. Confirm:
   `kubectl get runtimeclass` (does `windows-2022` exist?) and
   `kubectl -n $NS describe pod <blitzy-worker-…>` (events:
   `RuntimeClass … not found`, or `0/N nodes available: … node(s) didn't match`).
   Per-cluster fix is the `WINDOWS_RUNTIME_CLASS_NAME` env / Helm value, not code.

## Suggested debugging workflow

1. **Frame it.** One sentence: which component, which symptom, which env/cluster,
   which `execution_id`/`job_id`/`runner_id`/`client_id` if known.
2. **Pull evidence in parallel** (Datadog logs by service+id, `kubectl get/describe/
   logs` on the relevant pods, queue depths). Load Datadog skills first.
3. **Pattern-match** against "Known failure modes" — most incidents are a variant.
4. **Localize** to one channel (Redis command transport vs HTTP SCM vs K8s vs
   Postgres vs relay WS) and one repo. State the hypothesis.
5. **Confirm** with one decisive piece of evidence before naming a root cause.
6. **Propose the narrowest fix**, name its blast radius across repos, and never
   apply cluster mutations or Helm changes without explicit approval.

## Local dev note

Mediator runs locally via **Telepresence**; relay + workers stay in-cluster. Stack
traces with `/Users/m.makhzomi/...` paths are the **local** mediator, not the pod.
To reach a remote client's mediator HTTP API from a laptop, remember it connects
*outbound* to the relay — direct HTTP may require the relay path, a port-forward,
or Telepresence, not a plain public URL.
