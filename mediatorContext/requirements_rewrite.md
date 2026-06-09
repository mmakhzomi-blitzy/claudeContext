# Requirements for Rewrite — Mediator + Relay

**Scope**: Both mediator (customer-side) and relay (platform-side). Single replacement framework, written in Go.

**This doc is neutral** — capabilities and constraints only. Framework recommendation lives in `framework_evaluation.md`.

---

## 1. Goal and pain points

### Goal
Sustain **1000–1500 concurrent runners per customer** (today's choke point is ~250).

### Identified pain points the rewrite must address
1. **Throughput**: current Python + Socket.IO + gevent gunicorn pool tops out around 250 concurrent runners per mediator pod. Vertical scaling has hit a wall; horizontal scaling is blocked by C5/C6 (single WS connection per mediator).
2. **Multi-mediator scaling**: today there's no clean way to run multiple mediator pods for one customer — each would open its own WS connection to relay, causing routing ambiguity ("which mediator gets this request?").

### Explicitly NOT goals
- Fixing reconnection edge cases (e.g. "Invalid empty packet received") is **not** a rewrite driver. Reconnect must work, but it's not framed as a defect to chase.
- gRPC adoption is ruled out (CDP tunnel doesn't fit gRPC's stream semantics).
- Rewriting the SDK (`BlitzyClient` in `blitzy-utils-python`). The SDK should switch with a URL change only — see `contract_sdk.md` for what's frozen and the small envelope of changes allowed.

---

## 2. Capabilities the new system must provide

Each capability is `MUST` unless flagged. See `features_master.md` for the exhaustive list with file:line anchors. This section is a compressed checklist.

### 2.1 SDK-facing HTTP surface
- All endpoint paths under `contract_http_api.md` must work unchanged.
- Headers preserved: `x-client-key`, `X-Client-ID`, `X-Async`, `X-Relay-Request-ID`, `X-Correlation-ID`.
- Request/response JSON shapes preserved.
- Queue-on-miss semantics preserved: `202` with `X-Relay-Request-ID` when mediator offline + `X-Async: true`.

### 2.2 Mediator ↔ Relay transport
The new transport layer:
- May use a different protocol than Socket.IO.
- **MUST** support full-duplex binary frames (for CDP tunnel `tunnel_data`).
- **MUST** support a streaming HTTP-style request/response model (for `/control` request dispatch).
- **MUST** support a publish/subscribe / room-based fanout (for CDP tunnel multi-participant rooms).
- **MUST** survive transient TCP disconnects with automatic resume / re-registration.
- **SHOULD** support payloads up to 100 MB without forcing the client to chunk at the application layer (constraint M4).

### 2.3 Inbound store-and-forward
- Per-client request queue surviving mediator downtime up to N entries (current default 1000).
- Pending-response storage with TTL exceeding the max forwarding timeout (current 960s vs 900s).
- Distributed lock semantics for drain workers running across replicas.
- Multi-replica request/response correlation (today: `RedisPendingRequests` via Redis polling).

### 2.4 Outbound store-and-forward
- Mediator-side queue for in-flight outbound `WSHttpClient` calls when relay is offline.
- Drained on every successful reconnect.
- Synthetic 202 response to caller when queued (`{"queued": True}`).

### 2.5 CDP tunnel
- Room-based fanout (or equivalent multiplexing) — multiple connections per `tunnel:{runner_id}` channel.
- Three message types: connection lifecycle (`open`/`close`), binary data (`data`), HTTP-style discovery (`http`/`response`).
- Bidirectional `data` flow at low latency (CDP frames are interactive).
- Heartbeat / circuit-breaker: CDPProxy on consumer side probes the tunnel and fast-fails when broken. Tunnel must support a low-overhead liveness probe.

### 2.6 Worker / runner pod lifecycle
- K8s `Deployment` + optional `Service` + `HTTPRoute` (Gateway API v1) creation.
- Env injection into pod spec.
- `Affinity` / `nodeSelector` / `tolerations` propagated from chart values via configmap → mediator → `V1PodSpec`.
- Image pull secrets honored.
- Teardown removes Deployment + Service + HTTPRoute and archives command history.

### 2.7 Worker dispatch
- Per-runner queue addressable by `runner_id`.
- Shared results queue consumed by an in-mediator background worker.
- Result handler updates the postgres `command_executions` row.

### 2.8 Vault integration
- Customer secrets CRUD at Vault path `environment/{env_id}`.
- K8s SA-based authentication to Vault.
- In-process cache with explicit invalidation.

### 2.9 Observability
- Structured logs with `image_tag`, `client_id`, `correlation_id` global fields.
- Per-request correlation ID propagation through HTTP, WS, and worker env.
- All telemetry flows mediator → relay → forwarder (no direct customer-cluster → vendor SaaS egress).
- Deep health endpoints (`/health/connections`, `/health/queue` on relay) preserved.

### 2.10 Configuration
- All current env vars supported (see `consts.py` of mediator and relay).
- Helm chart values continue to work; chart may need minor template updates for new container args.

---

## 3. Non-functional requirements

| Requirement | Target | Today |
|---|---|---|
| Concurrent runners per customer | **1000–1500** | ~250 |
| Concurrent customers per relay process | TBD (research) | not measured; soft limit by gunicorn worker pool size |
| Request body max | 100 MB | 100 MB |
| Request response timeout | 15 min (configurable) | 15 min (`RELAY_FORWARD_TIMEOUT=900s`) |
| Mediator-side reconnect latency on relay restart | < 10s typical | ~5–30s (manual backoff: 1s → 5s capped) |
| Customer-cluster outbound: TLS to single relay endpoint | yes | yes |
| Multi-arch images | linux/amd64 + linux/arm64 | yes |
| Customer registry mirroring | required | manual today |

### Open questions for measurement (deferred to research phase)
- What's the per-process WS connection count ceiling for the chosen framework?
- Where does throughput cap on a single-mediator-pod design — CPU, GC, syscalls, or Redis IOPS?
- Cost of CDP tunnel passthrough at 1000 concurrent Chrome sessions?

---

## 4. The multi-mediator question (open architecture decision)

Currently, **one customer = one mediator pod**, which holds the WS connection to relay. Beyond ~250 runners that pod chokes. Scaling out to N pods per customer hits a routing problem: which pod gets which incoming SDK request?

Four candidate architectures. The rewrite must commit to one. Tradeoffs below; decision lives outside this doc (research phase informed by `framework_evaluation.md`).

### Option 1: Single beefy mediator (vertical scale)
Keep one mediator pod per customer. The new Go implementation alone, at the same hardware, hits the target throughput because Go is faster than Python+gevent.

| Pros | Cons |
|---|---|
| Simplest deployment | If 1500 runners still chokes one pod, no fallback |
| No multi-pod routing concerns | Single point of failure per customer |
| No state coordination | Vertical scaling limited by k8s node size |

### Option 2: Sharded mediators (consistent hashing)
N mediator pods per customer. Each pod owns a subset of `runner_id`s via consistent hashing. Relay knows the mapping and routes accordingly.

| Pros | Cons |
|---|---|
| Linear horizontal scaling | Relay must know mapping; per-pod routing in WS layer |
| Each pod's WS connection is independent | Re-sharding on pod restart causes runner-to-pod reassignment |
| Failures isolated to one shard | More complex teardown when runner moves between shards |

### Option 3: Broker + worker pods
One thin "connection broker" pod holds the WS connection(s). N stateless mediator pods sit behind it; broker dispatches incoming SDK requests via internal load-balanced HTTP/2 (or similar). Mediator pods talk to the broker for outbound calls.

| Pros | Cons |
|---|---|
| Mediator pods are truly stateless (Redis-only state) | Broker is a new component to design, build, and operate |
| Mediator scales independently of WS connection count | Broker becomes SPoF unless replicated; replication needs sticky-session or shared-state design |
| Customer cluster sees one stable WS connection | Two-hop internal request flow (extra latency, extra failure surface) |

### Option 4: Stateless mediator pool + Redis-as-source-of-truth
Any mediator pod can serve any request. Coordination via Redis (locks for runner lifecycle, etc.). Each pod opens its own WS connection to relay; relay sees N connections per customer but treats them as one fan-out destination.

| Pros | Cons |
|---|---|
| No new component (no broker) | N WS connections per customer; relay-side complexity for fanout |
| Pods scale freely behind PDB | "Which mediator gets this request" still requires routing logic, just pushed to relay |
| Failure of one mediator pod = transparent | More Redis round-trips per request |

### What "decision" looks like
The research phase should produce a single chosen option and update this doc with:
- The decision and reasoning
- Where the routing logic lives (which component knows mappings)
- HA design (replication, leader election if any, failover behavior)
- Cost model (how many pods does a 1500-runner customer need)

---

## 5. Transport-layer options for the rewrite

This is where the framework choice has the most leverage. Four candidate transports:

### Transport A: Native WebSockets (gorilla/websocket or coder/websocket)
- Single WS connection per mediator, custom framing protocol for `request` / `response` / `tunnel_data` / etc.
- Server: HTTP/2 + Upgrade to WS.
- Closest to today's Socket.IO model.

### Transport B: HTTP/2 server-push + chunked POST
- Each SDK request → HTTP/2 stream. Mediator pulls work via long-polling or server-sent events; pushes response via POST.
- Better backpressure than WS frames.
- CDP tunnel needs separate WS or HTTP/2 stream for binary bidirectional data.

### Transport C: HTTP/2 bidirectional streaming (akin to gRPC streams but using `Content-Type: application/octet-stream`)
- Both directions over HTTP/2 streams without WS upgrade.
- Most modern; framework support varies.

### Transport D: Mix — control-plane on HTTP/2 + tunnel on WS
- `/control` requests use HTTP/2 streams.
- CDP tunnel uses native WS (keeps WS where it's most needed).
- Decouples failure modes.

The framework comparison in `framework_evaluation.md` ranks candidates by which transports they support cleanly.

---

## 6. Hard constraints (carried from `features_master.md` §M)

1. Single customer-egress chokepoint — no direct customer-cluster to vendor SaaS.
2. HTTP/2 (or HTTP/1.1) over TLS — no custom L4 protocols.
3. Relay knows `client_id` per connection.
4. Body size up to 100 MB.
5. Request timeout up to 15 min.
6. Outbound queue persists across mediator restart.
7. RQ-on-Redis worker dispatch is the contract with `archie-client-worker`. Changing this requires worker-repo changes.
8. Customer's K8s namespace is the security perimeter.

---

## 7. What's frozen on the SDK side

See `contract_sdk.md`. Summary:

**Frozen** (rewrite must preserve):
- HTTP REST endpoint paths
- Request/response JSON shapes
- Header names and semantics
- Queue-on-miss flow (202 + `X-Relay-Request-ID` + poll endpoint)

**Minimal changes allowed** (a small SDK bump is acceptable for these):
- Adding optional headers (e.g. `X-Trace-Id`)
- Adding new optional config knobs (retry caps, transport timeouts)
- Bumping the relay URL (already user-configurable)
- Adding awareness of new error codes returned by relay

**Breaking changes prohibited**:
- Removing/renaming any current endpoint or field
- Changing auth header from `x-client-key` to anything else
- Changing the meaning of any existing status code

---

## 8. Migration path expected

The rewrite must support **both modes during transition**:
- Old SDK + Python relay+mediator (current production)
- Old SDK + Go relay+mediator (new system, URL switched)

Implications:
- The on-the-wire HTTP REST surface (SDK ↔ relay) is identical between old and new.
- The internal Socket.IO between relay and mediator can change freely — no customer dependency.
- During cutover, a customer is fully on one stack or the other; no straddling.

---

## 9. What the research phase should answer

Before committing to a framework or architecture:

1. Throughput baseline for each candidate framework at our request shape (small JSON requests, occasional 100 MB blobs, persistent WS connection).
2. CDP tunnel latency under each transport option (target: <50 ms added latency for in-region traffic).
3. Multi-pod coordination cost (Redis round-trips per request) under each multi-mediator option.
4. Operational cost of broker (Option 3) — does the simplification of stateless mediator pods justify a new component?
5. Memory / GC behavior under sustained 1500-runner load.

Each candidate framework's chapter in `framework_evaluation.md` cites the data we have today; the rest goes in the research bucket.
