# Store-and-Forward — Relay Queue & Connection Resilience

Canonical architecture for the WS relay's inbound store-and-forward system
and the mediator's outbound queue. Covers the full request lifecycle when
the mediator (or the relay itself) is temporarily offline.

> This document describes the **as-built** implementation. For the pre-
> implementation plan and test cases, see `WEBSOCKET_MIGRATION_PLAN.md`
> and `WEBSOCKET_MIGRATION_TEST_CASES.md`.

---

## 1. What It Solves

The SDK talks to the relay over HTTP. The relay talks to the mediator over
WebSocket (Socket.IO). The mediator talks to the backend over the same WS,
brokered back through the relay.

Two availability gaps must not drop requests:

| Gap | Direction | Mechanism |
|-----|-----------|-----------|
| Mediator offline | SDK → Relay → *Mediator* | **Inbound queue** — Redis Streams on relay, polled by SDK |
| Relay offline | *Relay* ← Mediator | **Outbound queue** — Redis list on mediator, drained on reconnect |

Neither layer blocks the caller's main thread indefinitely. The SDK
transparently polls a 202. The mediator returns a synthetic 202 and drains
in the background when the relay comes back.

---

## 2. Inbound Queue (Mediator Offline)

### 2.1 Request Flow

```
SDK                Relay (Flask + Socket.IO)            Mediator (WS)
 │                       │                                    │
 │ HTTP POST /path       │                                    │
 │ X-Client-ID: <cid>    │                                    │
 │ X-Async: true         │                                    │
 ├──────────────────────►│                                    │
 │                       │ registry.is_online(cid)? NO        │
 │                       │ XADD stream:requests:<cid> {req}   │
 │                       │ SET relay:pending:<rid> {pending}  │
 │                       │ SADD relay:drain:pending <cid>     │
 │ 202 Accepted          │                                    │
 │ X-Relay-Request-ID:   │                                    │
 │   <rid>               │                                    │
 │ {request_id, pending} │                                    │
 │◄──────────────────────┤                                    │
 │                       │                                    │
 │ GET /api/v1/queue/    │                                    │
 │   requests/<rid>      │                                    │
 ├──────────────────────►│                                    │
 │                       │ GET relay:pending:<rid> → pending  │
 │ 202 {status: pending} │                                    │
 │◄──────────────────────┤                                    │
 │                       │                                    │
 │     ... poll loop ... │                                    │
 │                       │                                    │
 │                       │   (mediator reconnects)            │
 │                       │◄───────────────────────────────────┤
 │                       │                                    │
 │                       │ DrainPool worker:                  │
 │                       │   XRANGE stream:requests:<cid>     │
 │                       │   socketio.emit("request", ...) ──►│
 │                       │                                  exec
 │                       │◄──── "request_response" ───────────┤
 │                       │   SET relay:pending:<rid> {done}   │
 │                       │   XDEL stream:requests:<cid> <id>  │
 │                       │                                    │
 │ GET .../requests/<rid>│                                    │
 ├──────────────────────►│                                    │
 │                       │ GET relay:pending:<rid> → done     │
 │ 200 {body, status,    │                                    │
 │      headers}         │                                    │
 │◄──────────────────────┤                                    │
```

### 2.2 Header Contract

| Header | Source | Purpose |
|--------|--------|---------|
| `X-Client-ID` | SDK → Relay | Target mediator (required on all relay calls) |
| `X-Async: true` | SDK → Relay | "Queue me if mediator is offline; don't 503." Opt-in |
| `X-Relay-Request-ID: <uuid>` | Relay → SDK (on 202) | **Marker that the 202 was queued by the relay**, not passed through from the mediator. SDK uses presence of this header to decide whether to enter the polling loop |

Mediator-originated 202s are passed through **without** `X-Relay-Request-ID`,
so the SDK treats them as real application responses.

### 2.3 Redis Key Layout

| Key | Type | Purpose | TTL |
|-----|------|---------|-----|
| `relay:requests:<client_id>` | Stream | Queued requests, FIFO | capped at `REQUEST_QUEUE_MAXLEN` (1000) |
| `relay:pending:<request_id>` | String (JSON) | Pending/completed response | `PENDING_TTL` (960s) |
| `relay:drain:pending` | Set | Client IDs needing a drain | — |
| `relay:drain:lock:<client_id>` | String (NX) | Distributed lock per client | short lease |
| `relay:registry:<client_id>` | Hash | `{sid, connected_at, last_heartbeat}` | — |

### 2.4 Endpoints

All under Flask blueprints on the relay:

| Method | Path | Purpose |
|--------|------|---------|
| `GET, POST, PUT, DELETE` | `/<path:path>` | Catch-all proxy (blueprint `relay_bp`) |
| `GET` | `/api/v1/queue/requests/<request_id>` | Poll for queued request's response |
| `DELETE` | `/api/v1/queue/<client_id>` | Purge client's queued requests |
| `GET` | `/api/v1/health/queue` | Drain pool + per-client queue stats |
| `GET` | `/api/v1/health/connections` | Connected mediators |

**Registration order matters.** The catch-all `/<path:path>` must be
registered **last** (`relay_bp` after `queue_bp`, `connections_bp`, `utils_bp`
in `main.py`). Otherwise the catch-all swallows the management endpoints.

Queue endpoints live under `/api/v1/queue/` (not `/queue/` or `/relay/queue/`)
because the GKE Gateway `HTTPRoute` matches `/relay/api/v1/*` and rewrites to
`/api/v1/*`. Endpoints outside that prefix are unreachable through the gateway.

### 2.5 Polling (SDK)

`blitzy_utils.blitzy_client._RequestsAdapter`:

- Reads `SERVICE_URL_RELAY` once in `__init__`, stored as `self._relay_url`.
  Fails fast with `ValueError` if `async_delivery=True` and the env var is
  unset — no silent fallback.
- On response: if status is 202 **and** `X-Relay-Request-ID` header is
  present → enter `_poll_until_complete(request_id)`.
- Polling: `GET {_relay_url}/api/v1/queue/requests/{id}` every
  `PollConfig.interval` seconds, with tenacity retry on
  `ConnectionError`/`Timeout`. Overall deadline 5 minutes.
- Terminal states: `completed` (return body+status), `failed` (raise).

### 2.6 Drain Pool

`src/services/drain_pool.py`:

- Fixed-size thread pool (`DRAIN_WORKER_COUNT`, default 4).
- Each worker loops:
  1. `SPOP relay:drain:pending` to claim a client to drain.
  2. `SETNX relay:drain:lock:<client_id>` to avoid two workers on the same client.
  3. Drain up to `DRAIN_BATCH_SIZE` (100) entries:
     - `XRANGE <stream> - + COUNT 1` — read oldest entry.
     - `socketio.emit("request", payload, room=<sid>)`.
     - Wait for response via `pending_requests.wait_for_response(rid)`.
     - On success: `XDEL <stream> <entry_id>`, persist response to
       `relay:pending:<rid>`.
     - On failure: **do not** XDEL — entry stays for next cycle.
  4. Release lock. If stream still has entries, `SADD` client back for
     another round.
- Triggered:
  - On mediator `on_connect` (control namespace).
  - From `relay.py` when a request is queued (so the drain doesn't wait for
    a reconnect if mediator is already online).

### 2.7 Why XRANGE+XDEL instead of consumer groups

The original design used `XREADGROUP` + `XACK`. Two problems:

1. Pending entries in the consumer group PEL blocked new `>` reads until
   resolved.
2. Acked entries were not deleted from the stream — they accumulated
   indefinitely (found 14 "orphaned acked" entries during debugging).

Plain `XRANGE` reads oldest-first, `XDEL` only happens on success. No
consumer group state to reason about. Failures naturally retry because the
entry isn't deleted.

---

## 3. Outbound Queue (Relay Offline)

### 3.1 Problem

When the relay is down, `WSHttpClient._call()` raises immediately. That
breaks in-flight pipelines — webhooks lost, job registration fails, secret
sync never fires. Some calls are fire-and-forget and can safely be deferred;
others (like a token fetch blocking a git clone) cannot.

### 3.2 Design

`src/services/ws_http_client.py`:

```python
def post(self, endpoint, data=None, headers=None, queueable=False):
    return self._call("POST", endpoint, body=data, headers=headers, queueable=queueable)

def _call(self, method, path, body=None, params=None, headers=None, queueable=False):
    payload = {...}
    try:
        response = self._sio.call("backend_request", payload, ...)
    except Exception:
        if queueable and self._redis:
            self._redis.lpush(OUTBOUND_QUEUE_KEY, json.dumps(payload))
            return WSResponse({"status": 202, "body": {"queued": True}})
        raise
```

- `queueable=True` → on failure, `LPUSH mediator:outbound_queue <json>` and
  return synthetic 202.
- `queueable=False` (default) → raise as before.
- `GET` is never queueable (always needs a response).

### 3.3 Caller Classification

| Caller | Method | Path | Queueable |
|--------|--------|------|-----------|
| `ghes.py` webhook forward | POST | `/v1/ghes/client/webhook/...` | yes |
| `blitzy_service.register` | PUT | `/v1/client/register` | yes (heartbeat) |
| `blitzy_service.register_job` | POST | `/v1/client/job/register` | **no** (default) — call must succeed before runner is usable, so it raises rather than queues |
| `blitzy_service.request_secret_sync` | POST | `/v1/client/environment/sync` | yes |
| `blitzy_service.get_job_info` | GET | `/v1/client/job/<id>` | no |
| `scm_service` token fetches | GET | token endpoints | no |

### 3.4 Drain on Reconnect

`ws_client.py::_on_control_connect` spawns a background thread that:

```
while True:
    raw = redis.rpop(OUTBOUND_QUEUE_KEY)
    if not raw: break
    try:
        sio.call("backend_request", json.loads(raw), ...)
    except Exception:
        redis.lpush(OUTBOUND_QUEUE_KEY, raw)  # put it back
        break                                  # retry on next reconnect
```

FIFO-ish: `LPUSH` newest at head, `RPOP` pulls oldest first. Failed
entries are pushed back to the head and retried on the next reconnect.

### 3.5 Stale SIO Reference

`WSHttpClient` holds a reference to the `WSClient` object, not the raw
`socketio.Client`. `_sio` is a property that returns `ws_client._sio`
every call — so after a reconnect (which creates a fresh `socketio.Client`),
all outbound calls automatically target the new client.

---

## 4. Response Chunking (Large Payloads)

Socket.IO has a per-frame size limit and a ping timeout that can be
exceeded during large payload transfers.

`ws_client._emit_response()`:

- If response body serialized ≥ 5 MB, emit as N × `response_chunk` events.
- Each chunk: `{request_id, sequence, total, payload}`.
- `time.sleep(0.1)` between emits to let ping/pong heartbeats process.

Relay `control.on_response_chunk()`:

- Buffers chunks in `_chunk_buffers[request_id]`.
- On final chunk, assembles and calls
  `pending_requests.set_response(request_id, full_response)`.
- `on_disconnect` clears any pending buffers for the disconnecting SID
  to avoid leaks.

`max_http_buffer_size = 100 MB` in `SocketIO(...)`.

`RELAY_FORWARD_TIMEOUT = 900s` (15 min) so the caller doesn't time out
during large chunked responses. `PENDING_TTL = 960s` (slightly longer)
keeps the pending key alive through the wait.

---

## 5. Multi-Worker Correlation (`RedisPendingRequests`)

### Problem
Pre-fix, pending-request state was an in-memory `dict[str, threading.Event]`
in `control.py`. Under multi-worker gunicorn, the worker that emitted the WS
request was often not the worker that received the response — the mediator
replied on whichever worker happened to own the Socket.IO session. Response
landed in a dead dict in another worker's memory. Requests timed out.

### Fix
`src/services/response_signal.py::RedisPendingRequests`:

```python
PENDING_KEY_PREFIX = "relay:pending:"
PENDING_TTL = 960

def set_response(self, request_id, response):
    redis.set(f"{PREFIX}{request_id}", json.dumps(response), ex=PENDING_TTL)

def wait_for_response(self, request_id, timeout):
    deadline = time.time() + timeout
    while time.time() < deadline:
        raw = redis.get(key)
        if raw:
            redis.delete(key)
            return json.loads(raw)
        time.sleep(0.05)   # cooperative under gevent
    return None
```

Any worker can now satisfy any other worker's pending wait. Redis is the
only shared state.

---

## 6. Connection Lifecycle (Mediator)

### 6.1 Fail-Fast

`WSClient.start()` uses `wait=True`; startup fails with `SystemExit` if the
initial connection doesn't succeed. Mediator won't boot without the relay.

### 6.2 Manual Reconnect

`reconnection=False` on `socketio.Client(...)`. Relying on python-socketio's
built-in reconnect with `threading` async mode triggers a known CLOSE-loop
bug (Issue #914).

Instead, `WSClient._schedule_reconnect()`:

- On every disconnect/error, instantiate a **fresh** `socketio.Client(...)`.
- Retry with backoff in a background thread.
- Auth rejection (401) → `SystemExit`. Other errors → keep retrying.

### 6.3 Auth Rejection

Relay raises `socketio.exceptions.ConnectionRefusedError("msg", {"status": 401})`
(not Python's builtin, which python-socketio scrubs). Mediator's
`connect_error` handler parses via `ConnectionRejection.from_connect_error()`.

### 6.4 on_connect Consolidates Everything

Originally `on_connect` + `on_register` were split. Socket.IO transport
upgrade (polling → WebSocket) sometimes dropped the original session before
the `register` event arrived → registry never updated → eternal reconnect
loop.

Now `on_connect` does auth + `join_room()` + `registry_service.register()`
atomically at handshake time. No separate `register` event.

---

## 7. Configuration

### Relay (`src/consts.py`)

| Constant | Default | Purpose |
|----------|---------|---------|
| `RELAY_FORWARD_TIMEOUT` | 900s | How long `proxy_request` waits for mediator |
| `PENDING_TTL` | 960s | TTL on `relay:pending:*` keys (constant; defined in `src/services/response_signal.py` — not env-configurable) |
| `RESPONSE_TTL_SECONDS` | 3600s | Legacy TTL (for completed-response readability) |
| `REQUEST_QUEUE_MAXLEN` | 1000 | Max Redis Stream length per client |
| `DRAIN_WORKER_COUNT` | 4 | Drain pool threads |
| `DRAIN_BATCH_SIZE` | 100 | Entries drained per client turn |
| `PING_TIMEOUT` | 600s | Socket.IO ping timeout |
| `PING_INTERVAL` | 60s | Socket.IO ping interval |
| `RELAY_USER_AGENT` | `blitzy-relay-tunnel/7x9K2mP4` | Injected on proxied backend requests |

### Mediator (env vars)

| Env var | Purpose |
|---------|---------|
| `SOCKETIO_PATH` | WS path (default `/relay/socket.io`) |
| `BLITZY_CLIENT_URL` | Included in `_build_auth` |
| `WORKER_CPU_REQUEST` / `WORKER_CPU_LIMIT` | Worker pod CPU resources |
| `WORKER_MEMORY_REQUEST` / `WORKER_MEMORY_LIMIT` | Worker pod memory resources |

Worker resources are **only** read from env — `kubernetes_service.py` no
longer falls back to API-payload config. Simplifies per-environment tuning.

### SDK (env vars)

| Env var | Purpose |
|---------|---------|
| `SERVICE_URL_RELAY` | Required when `async_delivery=True`; used for poll URL |

---

## 8. Operational Cheat Sheet

```bash
# Queue depth for a client
redis-cli XLEN relay:requests:<client_id>

# View queued entries
redis-cli XRANGE relay:requests:<client_id> - +

# Pending response state
redis-cli GET relay:pending:<request_id>

# Clients needing drain
redis-cli SMEMBERS relay:drain:pending

# Outbound queue (mediator side)
redis-cli LLEN mediator:outbound_queue
redis-cli LRANGE mediator:outbound_queue 0 -1

# Purge a client queue via API
curl -X DELETE "{relay_url}/api/v1/queue/<client_id>"

# Drain pool stats
curl "{relay_url}/api/v1/health/queue" | jq

# Connected mediators
curl "{relay_url}/api/v1/health/connections" | jq
```

---

## 9. Known Gaps

Tracked in `TASKS.md §13`:

- Manual drain trigger endpoint (`POST /api/v1/health/drain/<client_id>`).
- Client session diagnostics (SID cross-verification relay ↔ mediator).
- `BlitzyLogger` `%s` format support (Engine.IO uses positional args).
- `create_deployment` idempotent on 409.
- Gunicorn `--workers` bump to 4 (correctness already in place).
- Tenacity retry for non-queueable `WSHttpClient` calls on transient WS errors.
