# `/chat-runners` endpoints — spec

(For `archie-client-mediator`; companion to `PLAN.md` §3.4.)

Three endpoints in this family — `POST` (create, idempotent), `PATCH …/lease`
(extend TTL — heartbeat), and `DELETE` (immediate teardown). The
**annotation-based TTL** (`chat-worker.blitzy.io/expires-at`) is the
authoritative deadline; chat owns the policy via heartbeat, the mediator's
reaper enforces.

## `POST /api/v1/chat-runners` — create or look up

### Behaviour

1. **Idempotent on session-key label.** If a Deployment with label
   `chat-worker.blitzy.io/session-key=<chat_session_key>` exists and its
   `chat-worker.blitzy.io/expires-at` annotation is in the future, return the
   existing runner with HTTP 200 and `idempotent_hit=true`. Otherwise create
   a new Deployment and return HTTP 201 with `idempotent_hit=false`.
2. **Compatibility validated.** Before any k8s call, mediator calls
   `blitzy_utils.runner_ops.compatibility.assert_compatible(self.client_type,
   request.repo.installation_type)`. On mismatch returns HTTP 400 with
   `error_code=CHAT_WORKER_INCOMPATIBLE`. Defence-in-depth — chat already
   gates on the same function in `lease()`.
3. **Image is forced** to the mediator's `CHAT_WORKER_IMAGE` env var. Reject
   any `image_override` field in the payload with HTTP 400.
4. **Pod TTL** is set via the annotation
   `chat-worker.blitzy.io/expires-at: <epoch_seconds>`. The mediator's reaper
   enforces this. Default = `now + CHAT_WORKER_TTL_SECONDS` (21600 = 6 h).
   Mutable later via `PATCH …/lease`.
5. **Repo clone** is triggered by the worker's existing
   `CloneRepositoryStep` reading `EVENT_DATA`. The mediator
   serializes the `repo` block into `EVENT_DATA`.

### Request

```json
{
  "chat_session_key": "company:<uuid>:team:<uuid>:uid:<uuid>:project:<uuid>:git_project_repo_id:<uuid>:repo:<uuid>:repo_name:<str>:branch:<uuid>:tech_spec:<uuid>",
  "company_id": "<uuid>",
  "user_id": "<uuid>",
  "repo": {
    "repo_name": "<owner/repo>",
    "branch_name": "main",
    "user_id": "<uuid>",
    "git_project_repo_id": "<uuid>",
    "repo_id": "<uuid>",
    "head_commit_hash": "<sha or empty>",
    "server": "github.com",
    "installation_type": "GITHUB_ENTERPRISE_SERVER"
  },
  "ttl_seconds": 21600,
  "runner_config": {
    "cpu_request": "200m",
    "cpu_limit": "1000m",
    "memory_request": "512Mi",
    "memory_limit": "2Gi",
    "timeout_seconds": 6000
  },
  "metadata": {
    "source": "archie-service-chat",
    "session_kind": "chat"
  }
}
```

Notes:
- `chat_session_key` should be ≤ 253 chars (k8s label value limit) — mediator
  truncates to a sha256 prefix for the actual label/runner_id and stores the
  full key in an annotation.
- `ttl_seconds` is bounded: `min(max(900, ttl_seconds), 86400)` (15 min – 24 h).

### Response (201 Created or 200 OK on idempotent hit)

```json
{
  "runner_id": "chat-<sha256(chat_session_key)[:20]>",
  "chat_session_key": "<original>",
  "status": "DEPLOYING",
  "created_at": "2026-05-06T10:00:00Z",
  "expires_at": "2026-05-06T16:00:00Z",
  "idempotent_hit": false,
  "deployment_details": {
    "deployment_name": "chat-worker-<hash>",
    "namespace": "blitzy-workers",
    "image": "us-east1-docker.pkg.dev/.../archie-client-worker:<sha>"
  },
  "queue_details": {
    "queue_name": "blitzy-queue-chat-<hash>"
  },
  "ready_url": "/api/v1/runners/<runner_id>"   // poll for READY
}
```

### Errors

| Code | When |
|---|---|
| 400 `INVALID_REQUEST` | missing `chat_session_key`, `company_id`, or `repo` block |
| 400 `IMAGE_OVERRIDE_FORBIDDEN` | client tried to set `image_override` |
| 400 `CHAT_WORKER_INCOMPATIBLE` | `assert_compatible(client_type, repo.installation_type)` failed (e.g. GHES via BLITZY_SHARED). Body includes `details: {client_type, installation_type, reason}` |
| 401 / 403 | auth (same scheme as `/runners`) |
| 409 `LEASE_DRAINING` | a Deployment with this key exists but is in `Terminating` state — caller should retry after backoff |
| 429 `COMPANY_QUOTA_EXCEEDED` | per-company concurrent chat-worker limit (configurable; default 50) |
| 500 / 503 | k8s API failure |

## `PATCH /api/v1/chat-runners/<runner_id>/lease` — extend TTL

Extends the TTL on an existing chat-worker. Called by chat's heartbeat
coroutine every ~5 min during an active session, and once on
`/close_session` to start the 6h grace period.

### Request

```json
{ "ttl_seconds": 21600 }
```

Bounded the same as `POST`: `min(max(900, ttl_seconds), 86400)`.

### Behaviour

1. Look up Deployment by `runner_id` (label `runner-id=<runner_id>`).
2. Verify the caller's `company_id` matches the Deployment's
   `blitzy-job-id` (or equivalent) label. 403 on mismatch.
3. PATCH the annotation `chat-worker.blitzy.io/expires-at` to
   `now + ttl_seconds` via `patch_namespaced_deployment`. **Annotations
   are mutable; labels would not be.**
4. Return the new `expires_at`.

### Response (200 OK)

```json
{
  "runner_id": "chat-<hash>",
  "expires_at": "2026-05-06T22:00:00Z",
  "previous_expires_at": "2026-05-06T16:00:00Z"
}
```

### Errors

| Code | When |
|---|---|
| 404 `RUNNER_NOT_FOUND` | Deployment doesn't exist (already reaped) — caller should treat as "lease lost" and start a fresh session |
| 409 `RUNNER_TERMINATING` | Deployment is in `Terminating` state — same handling as 404 |
| 403 | `company_id` mismatch |

## `DELETE /api/v1/chat-runners/<chat_session_key>` — immediate teardown

Variant of `DELETE /runners/<runner_id>` keyed on the chat session.
Authorised only when the caller's `company_id` matches the runner's
`company_id` label. Returns same shape as `DeleteRunnerResponse`.

Used for explicit teardown (`/clear` from chat); not used by `/close_session`
(which lets the reaper handle deletion after the 6h grace period).

## Reaper

A background asyncio task in mediator startup, controlled by env vars
`CHAT_WORKER_REAPER_ENABLED` (default `true`) and
`CHAT_WORKER_REAPER_INTERVAL_SECONDS` (default 300):

```
every interval:
  for d in k8s.list_deployments(label_selector="chat-worker=true"):
    expires_at = int(d.metadata.annotations["chat-worker.blitzy.io/expires-at"])
    if now >= expires_at:
      runner_service.delete_runner(d.metadata.annotations["chat-worker.blitzy.io/runner-id"])
      log.info("reaped chat worker", runner_id=..., reason="ttl")
```

Add a Datadog gauge: `mediator.chat_worker.alive_count`,
`mediator.chat_worker.reaped_total`, `mediator.chat_worker.reap_errors`.

## Client adapter

In `blitzy_utils/blitzy_client.py`, add three methods (all follow the same
`get_client_url` + relay path as the existing `create_runner`):

```python
def create_chat_runner(self, company_id, chat_session_key, repo, *,
                       ttl_seconds=21600, runner_config=None) -> dict: ...

def extend_chat_runner_lease(self, company_id, runner_id, *,
                             ttl_seconds=21600) -> dict: ...

def delete_chat_runner(self, company_id, chat_session_key) -> dict: ...
```

Plus a new `RunnerSession` factory in `blitzy_utils/runner_ops/session.py`
that wraps these for `archie-service-chat`:

```python
class ChatRunnerSession(RunnerSession):
    """RunnerSession variant that:
       1. uses /chat-runners endpoints,
       2. validates compatibility before start,
       3. runs a heartbeat coroutine that extends the lease every
          HEARTBEAT_INTERVAL seconds,
       4. on stop(), patches a final 6h lease (does not call DELETE)."""
    def __init__(self, blitzy_client, company_id, chat_session_key, repo, ...): ...
    async def start_with_heartbeat(self, *, heartbeat_interval=300) -> None: ...
    async def stop(self) -> None: ...  # PATCH lease=now+6h, stop heartbeat
    async def force_terminate(self) -> None: ...  # DELETE
```
