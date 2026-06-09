# Connection Stability Research — Socket.IO Disconnects & The Path Forward

> **Status:** Research report, 2026-06-06. Author: research synthesis (two deep-research
> passes, 206 sub-agents total, adversarially verified) cross-checked against this repo's
> own docs (`contract_ws_protocol.md`, `STORE_AND_FORWARD.md`, `CDP_TUNNEL_PLAN.md`,
> `framework_evaluation.md`, `requirements_rewrite.md`, `CURRENT_STATE.md`).
>
> **Question asked:** Our mediator↔relay link constantly disconnects over the internet.
> Is this a known Socket.IO failure mode? How do Discord/Slack/Google Meet stay connected
> for hours, and what protocol does voice use? Is our connection multiplexed, and do
> HTTP/1.1/2/3 matter? What languages/runtimes are best? What is the most stable path forward?

---

## 0. TL;DR — the verdict

**Yes, this is a common, well-documented Socket.IO failure mode, and our own configuration
is the prime suspect.** The single most damning finding:

> `PING_TIMEOUT=600s` (with `PING_INTERVAL=60s`) is **~30× the library default**
> (25s interval / 20s timeout). A dead or half-open connection is therefore not detected
> for **~600–660 seconds**.

That ~600s blind spot is the **same "zombie window"** our own `CURRENT_STATE.md` already
describes for the sticky `is_online` flag and the relay forward-then-504 stalls. The long
timeout is an **active liability, not a safety margin.**

The deeper lesson from how Discord/Slack/voice apps actually work: **nobody keeps one
unbroken connection alive for hours.** They (a) detect death *fast* with a short heartbeat,
and (b) *re-establish with state resumed* (sequence numbers + RESUME). Our stack does the
opposite — detects death *slowly* (~600s) and has **no resume layer on the live channel**.

**Highest-leverage fix is config, not a rewrite:** retune the heartbeat + proxy idle
timeouts. That alone likely resolves the day-to-day pain and the zombie-`is_online` firefight.

---

## 1. Our current transport (as-built, from repo docs)

From `contract_ws_protocol.md` and `STORE_AND_FORWARD.md`:

| Property | Value |
|---|---|
| Protocol | Socket.IO over WebSocket (HTTP long-polling fallback present, "we don't rely on it") |
| Client lib | `python-socketio[asyncio_client]==5.16.1` |
| Server lib | `python-socketio>=5.14.0` + Flask-SocketIO + **gevent** worker class |
| Engine.IO | v4 |
| **Heartbeat** | **`PING_INTERVAL=60s`, `PING_TIMEOUT=600s`** |
| Multiplexing | Socket.IO **namespaces** `/control`, `/tunnel`, `/logs` over **one** HTTP/1.1 WebSocket |
| Max frame | 100 MB (`max_http_buffer_size`) |
| Reconnect | Manual; **fresh `socketio.Client()` every attempt** (workaround for python-socketio #914), exp backoff 1→5s |
| Resilience | Store-and-forward **queues** (inbound Redis Streams on relay; outbound Redis list on mediator) — but **no sequence/resume on the live socket** |

Documented workarounds that are really symptoms (from `contract_ws_protocol.md §7`):
`time.sleep(0.1)` between chunks (threading mode starves ping/pong), 6 MB chunking,
fresh-client-per-reconnect (#914), empty/invalid-packet Engine.IO errors.

---

## 2. Q1 — Is constant Socket.IO disconnect common? (Yes) + root causes

A recurring, first-class failure mode. Confirmed across **official GitHub issues**:

- **socketio/socket.io #5117** — *"client keeps disconnecting and reconnecting at a fixed
  interval because of ping timeout"* (reason=`ping timeout` even on websocket-only transport).
- Corroborated: **#5012** ("ping timeout after a few hours"), **#2769**, **#5116**,
  discussion **#5036**.
- **Flask-SocketIO #1065** — client disconnects ~30s after connect.
- `PING_TIMEOUT` is an **explicit named disconnect reason** in the framework — heartbeat
  failure is a built-in disconnect category.

### Most-cited root causes (all apply to us) and their fixes

| Root cause | Why it bites us | Fix |
|---|---|---|
| **Ping timeout mistuned too long** | 660s window ⇒ ~10 min to notice a dead socket (vs ~45s default) | **Lower `ping_timeout` to ~20–25s.** |
| **Proxy/LB idle timeout < heartbeat window** | nginx default `proxy_read_timeout` is **60s**; cloud LBs 30–350s — all **far below 660s** ⇒ intermediary silently reaps a *healthy* socket ⇒ surfaces as `transport close` / `ping timeout` | Raise each intermediary's idle timeout above the heartbeat window, **or** keep heartbeats shorter than the smallest idle timeout. nginx docs explicitly recommend periodic WS ping frames. |
| **Stuck on HTTP long-polling (no WS upgrade)** | Proxy not forwarding `Connection: upgrade` ⇒ Engine.IO error 3 / `TRANSPORT_MISMATCH`; slower + forces sticky sessions | Force **websocket-only transport**; set `proxy_set_header Upgrade/Connection`, `proxy_http_version 1.1`. |
| **Concurrency-model / event-loop starvation** | A blocking call starves the **gevent** hub ⇒ missed heartbeats ⇒ disconnect *independent of heartbeat config*. (In #1065 raising ping timeouts "didn't help"; the bug was a bad event loop.) | Audit blocking/un-monkey-patched calls. Our `time.sleep(0.1)`-between-chunks hack is a symptom of exactly this. |
| **Multi-worker needs sticky sessions + Redis backplane** | >1 worker w/o sticky ⇒ HTTP 400 `Session ID unknown`; Flask-SocketIO docs: gunicorn **can't run >1 worker** (no sticky support) | WebSocket-only transport removes the sticky requirement (single long-lived TCP conn). |

> ⚠️ **Refuted claim — do not rely on it:** the theory that ping/pong keeps succeeding *while*
> the connection is torn down (#5117) failed verification (1–2). The proxy-idle-timeout and
> long-`ping_timeout` causes are the solid ones.

---

## 3. Q3 — How Discord/Slack/Meet stay connected for hours; what voice uses

**Headline insight:** when you talk on Discord for hours, the connection is **not** literally
up the whole time. It silently fails over and resumes. Stability = **fast failure detection +
resume**, *plus* **a loss-tolerant media protocol** — both, not one unbroken pipe.

### The two-protocol split (the key architectural lesson)

| Plane | Transport | Why |
|---|---|---|
| **Control / signaling** (Discord Gateway, voice gateway, Slack Socket Mode) | **WebSocket** (reliable, ordered) + heartbeat + **sequence numbers** + **RESUME** | Control must be reliable & ordered |
| **Voice / media** (Discord voice, Google Meet) | **UDP / RTP**, encrypted Opus (WebRTC) | Media is loss-tolerant + latency-sensitive; TCP retransmission & head-of-line blocking *hurt* real-time audio — better to drop a packet than wait for it |

### Discord control-plane mechanics (what to copy)

- **Short heartbeat:** client sends Opcode 3 Heartbeat every interval from the server's
  Opcode 8 Hello — e.g. **~13.75s** (`heartbeat_interval` 13750ms); server replies Opcode 6 ACK.
  (Compare to our 600s.)
- **Sequence + resume:** since gateway v8, heartbeats carry `seq_ack` (last message received).
  On a drop, the client opens a **new** WebSocket and sends **Opcode 7 Resume**
  (`server_id`, `session_id`, `token`) ⇒ the server **replays missed messages** so the drop is
  invisible.
- **Failover with state reconstruction:** on a media-server (SFU) crash Discord restarts it
  "with minimal interruption (few dropped packets)" and *"state is reconstructed by the
  signaling component without any client interaction."*
- **NAT keepalive:** Discord routes media through its own relay so it **skips ICE/STUN/TURN**,
  but still sends **periodic pings to keep the firewall pinhole open** — short keepalives are
  mandatory to hold NAT/proxy state.

**Why voice "stays connected": BOTH** — UDP/RTP tolerates loss (transient loss ≠ disconnect),
*and* the reconnect/resume design hides the re-establishments. Slack Socket Mode and Google
Meet (WebRTC) follow the same control-vs-media reliability split. *(Discord is the
well-sourced exemplar; Slack/Meet specifics are slightly lower confidence.)*

---

## 4. Multiplexing & HTTP versions (your follow-up question)

Our CDP-tunnel-over-the-same-connection design is structurally fragile here:

| | Multiplexing | Head-of-line (HOL) blocking | Effect on our control + CDP |
|---|---|---|---|
| **HTTP/1.1 + one WebSocket (us, today)** | **App-level only** — Socket.IO namespaces over **one ordered TCP byte stream** | **Full TCP HOL blocking** | A CDP burst or one lost packet **stalls the control channel too**. Source of the `sleep(0.1)` hack. |
| **HTTP/2** (RFC 8441 WebSocket-over-HTTP/2, "Extended CONNECT") | **True protocol-level stream multiplexing** — control & CDP each a separate stream | **TCP-level HOL blocking remains** (single TCP conn) | Per-stream flow control & cancellation; a slow CDP stream no longer blocks control at the app layer — but a lost *packet* still stalls all streams at TCP layer. |
| **HTTP/3 / QUIC (+ WebTransport)** | **Independent streams over UDP** | **No transport HOL blocking** — loss on one stream stalls only that stream | Strongest fit for "control + lossy CDP burst on one connection"; also 1-RTT/0-RTT reconnect. |

**Two caveats for our case:**

1. **QUIC connection migration is NOT the silver bullet here.** It matters for *mobile clients*
   changing networks; our mediator↔relay link is **stable-IP datacenter-to-datacenter**.
   (The claim that migration is "transparent to the app" was **refuted** — it's client-initiated,
   needs path validation, and breaks under 4-tuple L4 load balancers.)
2. **Our own `requirements_rewrite.md` hard constraint:** *"HTTP/2 or HTTP/1.1 over TLS — no
   custom L4 protocols"* (customer firewalls only allow outbound HTTPS:443). **QUIC runs over
   UDP, which many corporate egress firewalls block/deprioritize.** So HTTP/3/WebTransport is
   the technically-best multiplexing answer but the **highest deployment risk**. **HTTP/2
   multiplexing (RFC 8441) is the pragmatic sweet spot.**

---

## 5. Language / runtime differentiators (Go is not treated as a constraint)

| Runtime | Unique strength for long-lived connections | Notes |
|---|---|---|
| **Erlang/Elixir (BEAM)** | **Documented best-in-class.** Preemptive per-process scheduling + isolated per-process heaps/GC ⇒ one connection can't starve/GC-stall others | WhatsApp **2M+ (peak 2.8M)/box**; Discord **11M concurrent on Elixir**, Rust only for hot-path NIFs |
| **Rust** | Lowest memory/conn; mature QUIC (**quinn**); "fearless concurrency" | The targeted accelerator Discord drops into |
| **Go** | Cheap goroutines + epoll netpoller; mature stdlib **HTTP/2** + `quic-go`; `gorilla/websocket` | Eliminates the python-socketio/gevent footgun class; matches our `framework_evaluation.md` (Chi + gorilla/websocket + stdlib http2) |
| **Java/Kotlin** | **Virtual threads (Loom, GA Java 21)** ~2× throughput vs platform threads for I/O-bound work | |
| **Node.js** | Where Socket.IO originated; single-thread event-loop caveats | |

> ⚠️ **Refuted claims to ignore:** the "Go collapses to 854 msg/sec vs Elixir" benchmark
> (cherry-picked, killed 0–3); "QUIC migration is transparent" (killed 0–3).

**Scale reality check:** BEAM's superpower is *millions* of connections. Our target
(`requirements_rewrite.md`) is **1,000–1,500 runners per customer** — trivial for *any* of
these runtimes. **Runtime choice is not our stability lever.** A Go rewrite buys *correctness*
(no Socket.IO/Engine.IO quirks), not raw capacity.

---

## 6. Stable alternatives for a server-to-server control channel

- **Raw WebSocket + short heartbeat + sequence/resume** — *recommended.* Strips Socket.IO's
  long-polling/upgrade/sticky/Engine.IO machinery (source of our documented failures) while
  staying firewall-friendly (HTTPS:443).
- **gRPC bidirectional streaming over HTTP/2** — excellent backend-to-backend (built-in
  keepalive, native multiplexing, no long-polling). **But** `requirements_rewrite.md` already
  ruled gRPC out *for the CDP tunnel*. Compromise: **gRPC for `/control`, native WS for `/tunnel`.**
- **NATS / message broker** — built-in PING/PONG liveness + dead-peer detection + pub/sub
  fan-out + store-and-forward. Better than a direct socket when you want decoupling/backpressure.
- **Centrifugo** — turnkey **sequence+resume**: per-channel offsets, replay missed messages on
  reconnect "as if the connection never dropped" (bounded by history TTL/limit). An off-the-shelf
  Discord-Opcode-7-Resume.
- **MQTT / SSE** — weaker fits (SSE is unidirectional; can't carry bidi control + tunnel).

---

## 7. 🎯 Recommendation — prioritized

### 1. Fix the resilience config FIRST — biggest win, near-zero effort, no rewrite
Likely resolves day-to-day pain *and* the zombie-`is_online` firefight:
- **Drop `PING_TIMEOUT` 600s → ~20–25s**, keep `PING_INTERVAL` ~25s. Detection ~660s → ~45s.
  *(Directly collapses the 600s zombie window in `CURRENT_STATE.md`.)*
- **Audit every intermediary idle timeout** on the mediator→relay path (GKE Gateway/
  `GCPBackendPolicy` 24h is fine; check nginx, any reverse proxy, NAT). Each must be
  **comfortably above** the heartbeat window — or keep heartbeats below the smallest one.
- **Confirm transport actually upgraded to WebSocket** (capture the Engine.IO handshake) — no
  silent long-polling.
- **Audit the gevent hub for blocking calls** (the `sleep(0.1)` hack is a smell).

### 2. Add sequence-number + session RESUME on the live channel (Discord Opcode 7 style)
We have store-and-forward *queues* but **no resume on the live socket** — a mid-flight drop
loses correlation. Add per-message sequence + replay-on-reconnect so drops become invisible.

### 3. Protocol (at rewrite time): drop Socket.IO for raw WebSocket, keep the two-plane split
Native WS over HTTP/2 (RFC 8441) gives true stream multiplexing for `/control` + `/tunnel`
without the Engine.IO footguns, within the HTTPS:443 / no-custom-L4 constraint. Treat
**HTTP/3/QUIC/WebTransport as "best multiplexing, highest firewall risk"** — validate against
real customer egress before committing (migration doesn't help our stable-IP link).

### 4. Runtime (lowest stability priority): Go is a fine, correctness-improving choice
Matches `framework_evaluation.md`; removes the python-socketio/gevent bug class. We do **not**
need BEAM/Rust for our connection counts — reserve that for a millions-of-connections target.

> **Bottom line:** Known Socket.IO failure mode; the 600s ping timeout is directly implicated;
> the **single highest-leverage fix is the heartbeat/timeout retune (#1), shippable without any
> rewrite.** Protocol and runtime changes are real but secondary to fast failure detection + resume.

---

## 8. Open questions / diagnostics to run

1. What are the **actual idle/read timeouts of every intermediary** on the path (cloud LB,
   nginx, reverse proxy, NAT/firewall)? Any below the 660s window is the proximate cause
   regardless of ping tuning.
2. Is the transport **actually upgrading to WebSocket end-to-end**, or silently stuck on
   long-polling? Capture the Engine.IO handshake.
3. Is the **gevent worker starved/blocked** (long synchronous calls, un-monkey-patched libs)
   so heartbeats are missed under load (the #1065 class of bug)?
4. For the forward path: build sequence+resume on raw WS ourselves (Discord style), adopt a
   backplane that provides it (Centrifugo offset recovery), or move `/control` to gRPC bidi —
   and what replay window / state-reload fallback is acceptable for control messages?

---

## 9. Caveats & confidence

- Socket.IO defaults (25s/20s) and nginx 60s reflect current docs; version-specific bugs
  (e.g. #5117 on 4.6.1) may be patched — confirm against our installed
  python-socketio/python-engineio/Flask-SocketIO versions.
- #1065's fix is asyncio-specific (2019); we use gevent, so the *fix* differs but the *lesson*
  (concurrency/event-loop/blocking bugs cause disconnects independent of heartbeat config) holds.
- Sticky-session & message-queue requirements are scoped to **multi-worker/replica** deployments
  and to **long-polling** transport — both sidestepped by websocket-only and/or a single relay process.
- No verified head-to-head quantitative benchmark of Socket.IO vs gRPC/Phoenix/MQTT/SSE exists in
  the claim set; the alternatives recommendation is synthesized from documented failure modes +
  Discord's design.
- **Refuted (do not use):** (a) QUIC migration "transparent to the app"; (b) the Elixir-vs-Go
  "854 msg/sec collapse" benchmark; (c) #5117's "ping succeeds while connection torn down" theory.

---

## 10. Sources (verified)

**Socket.IO / Flask-SocketIO / Engine.IO (primary):**
- Socket.IO troubleshooting — https://socket.io/docs/v4/troubleshooting-connection-issues/
- Engine.IO protocol — https://socket.io/docs/v4/engine-io-protocol/
- Using multiple nodes — https://socket.io/docs/v4/using-multiple-nodes/
- Flask-SocketIO API (defaults 25s/20s) — https://flask-socketio.readthedocs.io/en/latest/api.html
- Flask-SocketIO deployment — https://flask-socketio.readthedocs.io/en/stable/deployment.html
- python-engineio API — https://python-engineio.readthedocs.io/en/stable/api.html
- Issues: socketio/socket.io #5117, #5012, #2769, #5116, discussion #5036; Flask-SocketIO #1065;
  python-socketio #326

**Proxy / infra:**
- nginx WebSocket proxying — https://nginx.org/en/docs/http/websocket.html
- websocket.org heartbeat — https://websocket.org/guides/heartbeat/
- websocket.org timeout troubleshooting — https://websocket.org/guides/troubleshooting/timeout/
- GCP WebSocket LB — https://oneuptime.com/blog/post/2026-02-17-how-to-configure-load-balancing-for-websocket-applications-on-google-cloud/view

**Discord / voice / signaling (primary):**
- Discord voice connections — https://discord.com/developers/docs/topics/voice-connections
- Discord 2.5M concurrent voice via WebRTC — https://discord.com/blog/how-discord-handles-two-and-half-million-concurrent-voice-users-using-webrtc
- Discord gateway — https://docs.discord.food/topics/gateway
- Slack Socket Mode — https://docs.slack.dev/apis/events-api/using-socket-mode/

**Protocols / multiplexing (primary):**
- RFC 8441 (WebSocket over HTTP/2, Extended CONNECT) — https://www.rfc-editor.org/rfc/rfc8441
- RFC 9000 (QUIC), RFC 9114 (HTTP/3) motivation
- W3C WebTransport — https://www.w3.org/TR/webtransport/
- WebTransport vs WebSocket — https://websocket.org/comparisons/webtransport/
- gRPC vs WebSocket — https://websocket.org/comparisons/grpc/ ; gRPC core concepts — https://grpc.io/docs/what-is-grpc/core-concepts/
- quic-go connection migration — https://quic-go.net/docs/quic/connection-migration/
- NATS protocol (PING/PONG) — https://docs.nats.io/reference/reference-protocols/nats-protocol

**Runtimes / scale (primary + secondary):**
- WhatsApp scaling (Rick Reed) — https://www.slideshare.net/milkers/scaling-to-millions-of-simultaneous-connections-by-rick-reed-from-whatsapp
- WhatsApp architecture — https://highscalability.com/the-whatsapp-architecture-facebook-bought-for-19-billion/
- Discord Rust+Elixir 11M — https://discord.com/blog/using-rust-to-scale-elixir-for-11-million-concurrent-users
- Java virtual threads benchmark — https://github.com/ebarlas/project-loom-comparison

**Resume/replay backplane:**
- Centrifugo history & recovery — https://centrifugal.dev/docs/server/history_and_recovery

---

*Cross-references in this repo: `contract_ws_protocol.md` (transport contract, §7 "things the
rewrite can simplify"), `STORE_AND_FORWARD.md` (queues, §7 config incl. PING_*), `CDP_TUNNEL_PLAN.md`
(tunnel resilience/heartbeat/circuit-breaker), `framework_evaluation.md` (Go rewrite), `requirements_rewrite.md`
(transport constraints, no-custom-L4), `CURRENT_STATE.md` (the zombie/is_online firefight).*
