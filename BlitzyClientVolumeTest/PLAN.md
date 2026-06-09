# Volume test plan — Blitzy client stack

**Trigger:** Apr-2026 incident on a 40M-LOC repo. Cascading failures
spanning GKE (`MountVolume.SetUp` race + `BackoffLimitExceeded`),
Mediator + Relay 5xx, and worker polling collapse.

**Approach:** isolate each component, find its individual ceiling,
*then* compose. Local-first; cluster only after local results are
stable and reproducible.

---

## 1. Goals

For each component, produce three artefacts:

1. **A ceiling number** — "X concurrent runners" / "Y ops/sec" /
   "Z mediators online" before degradation.
2. **A saturation curve** — load vs latency p50/p95/p99 + error rate.
3. **A failure-mode label** — what specifically breaks first
   (Redis OOM? Flask thread pool exhausted? Socket.IO disconnects?
   K8s pod-creation throttle? RQ queue depth?).

**Out of scope (for now):** end-to-end integration test. We want to
isolate the components first; the integration test is a follow-up
once we have per-component baselines.

---

## 2. Test environments — local first, then cluster

### 2.1 Local (laptop) — primary

Each component runs in Docker containers on your laptop:

| Component | How |
|---|---|
| Redis | `redis:7-alpine` from the relay's docker-compose snippet, port 6379 |
| Mediator | `python main.py` with `LOCAL_DEVELOPMENT=true`, `MAIN_SERVER_URL=` (empty) |
| Relay | `python main.py` on port 8080, points to local Redis |
| Stubs | Stubbed `SERVICE_URL_ADMIN` / `SERVICE_URL_BACKEND` (small Flask app returning 200/JSON) |
| K8s API | Stubbed for the mediator's `kube_client_provider` — fake Pod create/delete returning success |

Why local first:
- Free, reproducible, no risk to shared infra
- Tight iteration loop (change code → restart → re-run)
- Lets us validate the test driver itself before wasting cluster time

### 2.2 Cluster (dev/stage) — secondary

Same drivers, different target URLs (env vars). All cluster tests must:
- Run in **off-hours** (or get explicit owner sign-off)
- Have a **single-command kill switch** (`make stop-load` — sends SIGINT to the driver)
- Be bounded by `--max-vus N` to prevent runaway
- Tag every emitted metric with `test_run_id` so we can correlate against cluster dashboards

---

## 3. Component 1 — Redis (isolation)

### 3.1 What we're testing

Redis sits underneath both Mediator and Relay. RQ uses it for command
queues; Relay uses it for the Socket.IO message queue + connection
registry + pending-request store. If Redis tips, everything tips.

### 3.2 Tools

- **`memtier_benchmark`** — synthetic mixed GET/SET load with
  configurable ratios, threads, pipeline depth. Industry standard.
- **`redis-cli --latency-history`** — passive latency monitor on a
  separate connection during the test.
- **`redis-cli MEMORY STATS`** + **`INFO`** — capture state at peak.

Skip `redis-benchmark` (built-in but limited).

### 3.3 Scenarios (run in order)

| ID | Name | Workload shape | Goal |
|---|---|---|---|
| **RD-1** | Baseline GET/SET | 90/10 GET/SET, 4 threads, 50 conns, 30s | Validate setup, get a sane baseline |
| **RD-2** | RQ queue shape | LPUSH + BRPOP loop simulating RQ's queue consumption | Mirrors mediator's actual usage |
| **RD-3** | Socket.IO message queue | PUBLISH + SUBSCRIBE storm; thousands of pub/sub channels | Mirrors relay's per-mediator routing |
| **RD-4** | Memory pressure | Continuously add keys until `maxmemory` policy triggers eviction | Find the size where eviction or OOM kills consumers |
| **RD-5** | Connection storm | Open 10K → 50K short-lived connections from N workers | Find the connection limit (often the binding constraint, not RPS) |
| **RD-6** | Mixed realistic | RQ + Pub/Sub + GET/SET concurrently at production-shaped ratios | Validates that no single workload starves others |

For each, capture: ops/sec, p50/p95/p99 latency, max memory, eviction
count, connected clients, rejected connections.

### 3.4 Ceiling outputs

By the end of RD-1..6 you should have:

- "Redis sustains **X ops/sec** at p99 < 5ms"
- "At **Y connected clients** new connections start being rejected"
- "At **Z MB used_memory** with current eviction policy, eviction
  rate is N/sec"
- "RQ queue depth where BRPOP starvation begins"

These numbers feed every downstream test as **bounds** — "the
mediator test must not exceed Redis ceiling, otherwise we can't
attribute failure to the mediator."

---

## 4. Component 2 — Mediator (isolation, mixed payloads)

### 4.1 What we're testing

The mediator exposes two distinct operation classes:

- **Command-class** (cheap, frequent, latency-sensitive)
  - `POST /api/v1/jobs/{job_id}/commands` — submit a command
  - `GET /api/v1/commands/{exec_id}/status` — poll status
  - `POST /api/v1/jobs/{job_id}/restart-session` — control signal

- **Runner-class** (expensive, infrequent, throughput-sensitive)
  - `POST /api/v1/runners` — provision K8s pod + Redis queue
  - `GET /api/v1/runners/{job_id}` — readiness poll (long-lived)
  - `DELETE /api/v1/runners/{job_id}` — cleanup

The user's specific request: **mixed-payload tests with varying
command:create ratios** to find the breaking point of each in
isolation and the interaction between them.

### 4.2 Tool

**k6** with weighted scenarios. Reasons:
- Native support for *scenarios* with independent VU pools, ramp
  shapes, and per-scenario thresholds. This maps 1:1 to "80%
  commands / 20% creates."
- JavaScript test definitions are easy to version + diff in PRs.
- Native Prometheus / InfluxDB output for our dashboards.
- Single binary; runs locally and in cluster identically.

Locust is the alternative (Python-native, easier for our team) — but
k6's mixed-scenario primitive is materially better for what we want.
Pick one and stick.

### 4.3 Scenarios

Six runs, each producing one curve. Each scenario specifies a target
**total RPS** held constant across the ratio sweep so we can compare
fairly.

| ID | Commands : Creates | Total target RPS | Goal |
|---|---|---|---|
| **M-1** | 100 : 0 | 50 → 200 → 500 RPS (ramp) | Pure-command ceiling |
| **M-2** | 0 : 100 | 0.5 → 2 → 5 RPS (ramp; creates are slow) | Pure-create ceiling |
| **M-3** | 80 : 20 | 50 → 150 RPS | The "primary" mix the user requested |
| **M-4** | 50 : 50 | match M-3 RPS | Equal split — does create cost dominate? |
| **M-5** | 20 : 80 | match M-3 RPS | Create-heavy — expect early saturation |
| **M-6** | Spike scenario | Constant 100 RPS commands + sudden burst of 20 creates over 10s | Replicates the production failure shape |

For each scenario, capture:
- Per-endpoint p50/p95/p99 latency
- Error rate split by class: 4xx, 5xx, timeout
- Mediator process: CPU, RSS, gunicorn worker count
- Redis: ops/sec, connection count
- Faked K8s API: call count, latency
- RQ queue depths

### 4.4 Local stubs

The mediator depends on:
- **Redis** — real, local
- **K8s API** — needs a stub. Use `kube_client_provider`'s injection
  point to swap in a fake that records calls and returns success
  after a configurable delay (default 2s to mimic real Pod create).
- **Blitzy backend** (`SERVICE_URL_ADMIN`, etc.) — small Flask stub
  app returning canned JSON for `/v1/auth/*`, `/v1/projects/*`.
- **Relay (Socket.IO)** — leave `MAIN_SERVER_URL` empty; mediator
  serves direct HTTP, no WS dependency in local mode.

### 4.5 Ceiling outputs

- "Pure commands: mediator handles **N RPS** before p95 > 200ms"
- "Pure creates: mediator handles **M concurrent provisions** before
  K8s API stub queue depth grows unbounded"
- "At 80/20 mix: total throughput is X RPS; first sign of
  degradation is..."
- "First failure mode: gunicorn thread pool / RQ queue saturation /
  Redis connection cap / memory growth / something else"

### 4.6 Why this matters for the cluster

Local tests give us a clean failure-mode signature ("first 5xx is
caused by gunicorn thread pool exhaustion"). When we run the same
scenario against the cluster, we compare: same signature → mediator
is the constraint; different signature → something else in the path
(real K8s, real Redis, real Backend) is the constraint.

---

## 5. Component 3 — Relay (isolation, multi-mediator)

### 5.1 What we're testing

Relay is a **stateless HTTP→WebSocket proxy** that routes per
`X-Client-ID`. It supports:

- N concurrent WebSocket-connected mediators (each in its own
  Socket.IO room)
- HTTP callers (SDK, admin) hitting `/<path>` with `X-Client-ID` —
  request gets routed to that mediator's room
- Async queueing in Redis Streams when a mediator is offline
- A drain pool of workers (default `DRAIN_WORKER_COUNT=4`) feeding
  queued requests back to mediators when they reconnect

The user's specific request: **load with multiple mediators**, since
that's what the relay is designed for.

### 5.2 Tools — two-part driver

This is the most complex test setup. Two pieces:

1. **Fake mediator simulator** — Python script using
   `python-socketio` client, connects to relay's `/control`
   namespace with a unique `X-Client-ID`, registers (`api_key`),
   joins room, replies to `request` events with synthetic delays.
   Spawn N of these as background processes.

2. **HTTP traffic generator** — k6 hitting relay's `/<path>` with
   `X-Client-ID` rotated across the connected mediators. Optionally
   sets `X-Async: true` to test the queueing path.

Why not just k6 for both? k6's WebSocket support is for raw WS, not
Socket.IO (which has its own protocol layer). The fake mediator must
speak Socket.IO; Python is easier and we can reuse the mediator's
own Socket.IO client code.

### 5.3 Scenarios

| ID | Name | Setup | Goal |
|---|---|---|---|
| **R-1** | Single mediator baseline | 1 fake mediator + 50 RPS HTTP | Establish the simplest case; confirm round-trip latency |
| **R-2** | 10 balanced mediators | 10 fakes + 500 RPS HTTP, X-Client-ID round-robin | Low-mediator-count baseline |
| **R-3** | 100 balanced mediators | 100 fakes + 5K RPS HTTP, round-robin | Stress the per-mediator-room routing |
| **R-4** | Heterogeneous load | 50 fakes; 5 receive 80% of HTTP, 45 receive 20% | Tests if "hot" mediators starve the rest |
| **R-5** | Mediator churn | Continuous load while 10% of mediators disconnect/reconnect every 30s | Tests connection registry + Redis Stream replay |
| **R-6** | Offline-queue replay | All mediators offline; 1K async requests posted; mediators reconnect; measure drain time | Tests the store-and-forward path explicitly |
| **R-7** | Per-mediator overload | 1 mediator, 1K RPS at it specifically | Find the per-mediator-room limit |

Capture per scenario:
- p50/p95/p99 round-trip latency from HTTP caller's view
- Redis: pub/sub channels, message rate, Stream entries per client_id
- Relay process: gevent worker count, memory, Socket.IO connected count
- Drain pool depth, drain rate
- WebSocket disconnect events (these are silent failures; they need to be counted)

### 5.4 Ceiling outputs

- "Relay sustains **X mediators** connected concurrently"
- "Relay routes **Y RPS** end-to-end with mediators echoing in Z ms"
- "First failure mode: gevent context-switch starvation / Redis
  pub/sub backlog / Socket.IO ping timeout / drain pool starvation"
- "Offline-queue replay rate: N requests/sec/client at full saturation"

---

## 6. Common workload patterns

Used across all components. Pick the right shape per scenario:

| Pattern | Shape | Use for |
|---|---|---|
| **Constant rate** | Hold X RPS for N minutes | Steady-state baseline; SLO regression |
| **Ramp** | 0 → target over T, hold, ramp down | Find ceiling without spike artefacts |
| **Spike** | Steady → sudden burst → steady | Reproduce production failure shapes |
| **Soak** | Constant rate for hours | Find slow leaks (memory, connections) |
| **Stress-to-failure** | Ramp until errors > threshold, capture state at break | Failure-mode characterisation |

---

## 7. Metrics + dashboards

### 7.1 What to capture (every test)

- **From the driver** (k6/memtier): RPS, p50/p95/p99 latency, error
  rate by class, total requests
- **From the target service**: process-level CPU/RSS, request count,
  error count, custom internal counters (RQ queue depth, gevent
  worker pool, Socket.IO connected count, etc.)
- **From the substrate**: Redis ops/sec, mem, connections; faked K8s
  API latency; Postgres connections (if applicable)

### 7.2 Local dashboard

For laptop runs:
- Run a local Prometheus + Grafana via docker-compose (5-minute
  setup). All targets and the k6 Prometheus output point at it.
- Pre-built Grafana dashboards (one per component) live in
  `metrics/dashboards/*.json`.

### 7.3 Cluster dashboard

For cluster runs:
- Push k6 metrics to the existing observability stack (Datadog or
  whatever the team uses).
- Tag every metric with `test_run_id`, `component`, `scenario_id`.

### 7.4 Output artefact per run

Every test run produces:
- Raw k6/memtier output (CSV)
- Captured Grafana / dashboard snapshot (PNG or share link)
- A one-pager `RUN-<test-id>.md` with: scenario, parameters, ceiling
  number, observed failure mode, dashboard link

These accumulate in `BlitzyClientVolumeTest/runs/`.

---

## 8. Safety + blast-radius rules

**Local:** unconstrained. It's your laptop.

**Cluster (dev):**

1. **No production traffic generators** — never point a load test at a
   shared production service.
2. **Kill switch** — every driver script must respond to SIGINT
   instantly and shut down all VUs. Test this before starting.
3. **Concurrency cap** — k6 `--max-vus 500` minimum cap; raise
   deliberately. Same for fake-mediator process count.
4. **Time cap** — hard timeout on every run (`--duration 30m` max
   unless explicitly running a soak).
5. **No write-side surprises** — runner-creation tests in cluster
   must use `dry-run=true` flag (or a known-safe namespace) to avoid
   provisioning real K8s pods that consume real resources.
6. **Notify before running** — post in the team channel: scenario,
   target service, expected duration, kill-switch command.
7. **Post-run cleanup** — drain Redis test keys, delete fake k8s
   pods, archive metrics, restore quotas.

---

## 9. Order of execution

1. **Redis local (RD-1..6)** — cheap, isolates the substrate.
2. **Mediator local (M-1..6)** — depends on Redis being characterised.
3. **Relay local (R-1..3)** — start small, build the simulator.
4. **Relay local (R-4..7)** — full multi-mediator + churn.
5. **Cluster Redis (RD-1..3 only)** — verify cluster Redis matches
   local within 20% on the same scenarios.
6. **Cluster Mediator** — same scenarios, real K8s API. Compare
   failure-mode signatures.
7. **Cluster Relay** — only after mediator results are stable.
8. **(Future) Integration test** — full pipeline at scale, once
   per-component ceilings are known.

Stop and discuss after each major step. Each scenario produces a
result that may change priorities for the next step.

---

## 10. Open questions for the team (to settle before we start)

1. **k6 vs Locust** — k6 by default per §4.2; flag if there's an
   existing Locust investment we should reuse.
2. **Cluster owner / sign-off** — who approves cluster runs?
3. **Observability backend** — Datadog, Grafana Cloud, or stand up
   our own? Affects §7.
4. **Image pinning** — every cluster run must use a digest-pinned
   image (the Apr-2026 incident showed `:latest` repushed
   mid-test). Where does the digest live?
5. **Stub fidelity** — the local K8s API stub is a coarse
   approximation. How important is real-Kubernetes behaviour for the
   mediator runner-creation tests? May push us straight to cluster.
6. **Existing baselines** — anything from past load tests we should
   diff against?
