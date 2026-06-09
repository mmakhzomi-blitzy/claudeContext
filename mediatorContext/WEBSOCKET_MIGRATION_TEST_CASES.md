# WebSocket Migration — Test Cases

Comprehensive test cases for the WebSocket relay, tunnel transport, and observability infrastructure. Organized by component and failure scenario.

---

## 1. Relay Service (archie-service-relay)

### 1.1 Connection Lifecycle

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 1.1.1 | Mediator connects to relay | Mediator starts with `WS_RELAY_URL` set. Socket.IO client connects with `api_key` in auth. | Relay authenticates, assigns socket to `company_id` room, creates entry in Redis connection registry. |
| 1.1.2 | Mediator reconnects after disconnect | Kill mediator's network for 30s, restore. | Socket.IO auto-reconnects. Redis registry updated with new `sid`. Pending requests resume. |
| 1.1.3 | Mediator connects with invalid API key | Connect with wrong `api_key`. | Relay rejects connection. `connect_error` event fired on client. No registry entry created. |
| 1.1.4 | Mediator connects with expired API key | Connect with revoked/expired key. | Same as 1.1.3 — rejected with auth error. |
| 1.1.5 | Duplicate mediator connection (same company) | Two mediators connect with same `company_id`. | Second connection replaces first in registry. First connection receives disconnect. |
| 1.1.6 | Mediator graceful disconnect | Mediator calls `sio.disconnect()`. | Registry entry removed. Relay returns 503 for subsequent SDK requests to that company. |
| 1.1.7 | Mediator ungraceful disconnect | Kill mediator process. | Relay detects via heartbeat timeout (~25s). Registry entry removed. |
| 1.1.8 | Consumer connects to `/tunnel` namespace | Consumer's `TunnelTransport` connects with `api_key`. | Authenticated, assigned to `company_id` room on `/tunnel` namespace. |

### 1.2 HTTP API — `/relay` Endpoint

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 1.2.1 | SDK request routed to mediator | `POST /relay` with `X-Company-ID: acme-corp` and valid runner create payload. | Relay looks up `acme-corp` socket, forwards via `sio.call()`, returns mediator's response as HTTP. |
| 1.2.2 | SDK request — mediator offline | `POST /relay` with `X-Company-ID` for a disconnected mediator. | HTTP 503 with `{"error": "Mediator offline"}`. |
| 1.2.3 | SDK request — unknown company | `POST /relay` with `X-Company-ID` not in registry. | HTTP 503 with `{"error": "Mediator offline"}`. |
| 1.2.4 | SDK request — missing header | `POST /relay` without `X-Company-ID`. | HTTP 400 with `{"error": "X-Company-ID header required"}`. |
| 1.2.5 | SDK request — mediator timeout | Mediator takes > 30s to respond. | Relay returns HTTP 504 Gateway Timeout. |
| 1.2.6 | SDK request — large payload | Submit command with 500 KB stdout. | Relay forwards and returns successfully (within 10 MB limit). |
| 1.2.7 | SDK request — payload exceeds limit | Send 15 MB payload. | Relay rejects with HTTP 413 Payload Too Large. |
| 1.2.8 | Concurrent SDK requests to same mediator | 10 simultaneous `POST /relay` requests for same company. | All routed to same mediator socket. All return correct responses (no cross-talk). |
| 1.2.9 | Concurrent SDK requests to different mediators | Requests for 5 different companies in parallel. | Each routed to correct mediator. No cross-contamination. |

### 1.3 Health API

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 1.3.1 | Health check — basic | `GET /v1/health-check` | `{"OK": true}`, 200. |
| 1.3.2 | Connection health | `GET /health/connections` | Lists all connected mediators with `company_id`, status, uptime, socket_id. |
| 1.3.3 | Tunnel health | `GET /health/tunnels` | Lists active tunnels per company with `runner_id`, `port`, `worker_connected`, `frames_forwarded`. |

### 1.4 Multi-Replica

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 1.4.1 | Cross-replica routing | Mediator connected to relay replica A. SDK request hits relay replica B. | Redis adapter routes message from B to A. Mediator receives request, response flows back through B to SDK. |
| 1.4.2 | Replica failure | Kill one relay replica. Mediators connected to it disconnect. | Mediators reconnect to surviving replica. Redis registry updated. No requests lost (SDK retries on 503). |
| 1.4.3 | Replica scale-up | Add a third relay replica. | New connections may land on new replica. Existing connections unaffected. Cross-replica routing works for all three. |

---

## 2. Tunnel Transport (Consumer → Relay → Mediator → Worker)

### 2.1 Tunnel Lifecycle

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 2.1.1 | Open tunnel | `TunnelManager.open(remote_port=9222, local_port=9222)` | `TunnelProxy` binds `localhost:9222`. No connection to worker yet (lazy). |
| 2.1.2 | First connection through tunnel | chrome-devtools-mcp connects to `localhost:9222`. | Proxy accepts TCP connection, generates `connection_id`, sends `{type: 'open'}` through relay to mediator. Mediator opens TCP to `chrome-svc-{runner_id}:9222`. |
| 2.1.3 | Data flows through tunnel | chrome-devtools-mcp sends CDP command. | Frame flows: proxy → transport → relay → mediator → worker Chrome. Response flows back same path. |
| 2.1.4 | Multiple connections on same tunnel | chrome-devtools-mcp opens connections to `/devtools/browser/...` and `/devtools/page/...`. | Each gets unique `connection_id`. Mediator opens separate TCP connections for each. No cross-talk. |
| 2.1.5 | Close tunnel | `TunnelManager.close_tunnel(remote_port=9222)` | Proxy sends `{type: 'close'}` for all active connections. Mediator closes TCP connections to worker. Proxy unbinds local port. |
| 2.1.6 | Close all tunnels | `TunnelManager.close()` | All proxies stopped, all connections closed, transport disconnected. |
| 2.1.7 | Multiple tunnels simultaneously | Open tunnels for ports 9222, 3000, 5432. | Three local ports bound. Each routes to correct worker port. Independent lifecycle. |
| 2.1.8 | Tunnel with port mapping | `open(remote_port=9222, local_port=19222)` | Binds `localhost:19222`, forwards to worker `9222`. |

### 2.2 Tunnel Data Integrity

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 2.2.1 | Small payload | Send 100-byte CDP command. | Arrives at worker Chrome unmodified. Response returns unmodified. |
| 2.2.2 | Large payload (screenshot) | Trigger `Page.captureScreenshot` returning 3 MB base64. | Full screenshot data flows back through all hops. No truncation, no corruption. |
| 2.2.3 | Max payload | Send 9.5 MB data through tunnel. | Succeeds (within 10 MB limit). |
| 2.2.4 | Oversized payload | Send 15 MB data through tunnel. | Rejected. Error frame sent back to consumer. Connection not dropped. |
| 2.2.5 | Binary data | Send raw binary (e.g., image bytes) through tunnel. | Binary preserved through all hops. No encoding issues. |
| 2.2.6 | Rapid sequential messages | Send 100 CDP commands in quick succession. | All arrive in order. All responses match correct request IDs. |
| 2.2.7 | Bidirectional simultaneous | Consumer sends command while Chrome pushes an event. | Both frames delivered correctly. No deadlock, no frame mixing. |

### 2.3 Tunnel Message Protocol

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 2.3.1 | Open message | Emit `{type: 'open', runner_id, port, connection_id}` | Mediator opens TCP to worker. Ack returned. |
| 2.3.2 | Data message | Emit `{type: 'data', runner_id, port, connection_id, data}` | Data forwarded to correct worker connection. |
| 2.3.3 | Close message | Emit `{type: 'close', runner_id, port, connection_id}` | Mediator closes specific TCP connection. Ack returned. |
| 2.3.4 | Error message (from mediator) | Mediator can't connect to worker. | Returns `{type: 'error', runner_id, port, connection_id, data: 'Connection refused'}`. |
| 2.3.5 | Invalid runner_id | Send tunnel message with non-existent `runner_id`. | Error frame returned: `"Runner not found"`. |
| 2.3.6 | Invalid port | Send tunnel message to port not exposed by worker. | Error frame returned: `"Connection refused"`. |
| 2.3.7 | Missing fields | Send tunnel message without `connection_id`. | Error frame returned: `"Missing required field: connection_id"`. |

---

## 3. Failure Scenarios — Per Hop

### 3.1 Hop 1: Consumer → Relay

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 3.1.1 | Relay unreachable on connect | Start consumer with relay URL pointing to dead host. | `TunnelTransport.connect()` raises connection error. Consumer handles gracefully (falls back or surfaces error). |
| 3.1.2 | Relay drops mid-session | Kill relay while tunnel is active. | Socket.IO fires `disconnect` event. Auto-reconnect starts. Consumer logs `tunnel_transport_disconnected`. |
| 3.1.3 | Relay reconnect succeeds | Relay comes back after 10s. | Socket.IO reconnects. `TunnelTransport` re-sends `{type: 'open'}` for all active tunnels. Tunnel resumes. |
| 3.1.4 | Relay reconnect fails (prolonged outage) | Relay down for 5 minutes. | Socket.IO retries with exponential backoff. Consumer logs each attempt. `TunnelProxy` returns errors to local clients. |
| 3.1.5 | Network latency spike | Introduce 2s latency on consumer → relay. | Tunnel still works but slow. Metrics show elevated `consumer.send_latency`. |
| 3.1.6 | Partial network failure | Consumer can reach relay but relay can't reach mediator. | Consumer sends succeed. Relay returns error frames. Consumer surfaces error to local clients. |

### 3.2 Hop 2: Relay → Mediator

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 3.2.1 | Mediator disconnects while tunnel active | Kill mediator while CDP session in progress. | Relay detects disconnect. Sends `{type: 'error'}` to consumer for all active tunnels on that mediator. Consumer closes local connections. |
| 3.2.2 | Mediator reconnects | Mediator restarts and reconnects to relay. | Registry updated. Consumer re-opens tunnels. Mediator opens fresh TCP connections to workers. |
| 3.2.3 | Tunnel frame for offline mediator | Consumer sends tunnel frame, but mediator just disconnected. | Relay returns `{type: 'error', data: 'Mediator offline'}`. |
| 3.2.4 | Relay routes to wrong mediator | Bug test — verify routing correctness. | Each frame's `company_id` (from socket session) matches the target mediator. Assert no cross-tenant routing. |

### 3.3 Hop 3: Mediator → Worker Service

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 3.3.1 | Worker service not created yet | Tunnel open request for runner whose K8s Service doesn't exist. | `TunnelForwarder` gets DNS resolution failure. Returns `{type: 'error', data: 'Service not found'}`. |
| 3.3.2 | Worker pod not ready | K8s Service exists but pod hasn't started. | TCP connection refused. Returns `{type: 'error', data: 'Connection refused'}`. |
| 3.3.3 | Worker pod OOMKilled | Worker pod dies while tunnel active. | TCP connection reset. `TunnelForwarder` read loop exits. Sends `{type: 'close'}` back to consumer. |
| 3.3.4 | Worker pod rescheduled | K8s reschedules pod to different node. | Old TCP connection drops. Sends `{type: 'close'}`. Consumer re-opens tunnel → new TCP to new pod. |
| 3.3.5 | Multiple workers, correct routing | Two runners active: runner-A and runner-B. | Tunnel for runner-A routes to `chrome-svc-runner-A:9222`. Tunnel for runner-B routes to `chrome-svc-runner-B:9222`. No cross-talk. |
| 3.3.6 | K8s network policy blocks connection | Network policy prevents mediator → worker traffic on port 9222. | TCP timeout. Returns `{type: 'error', data: 'Connection timed out'}`. |

### 3.4 Hop 4: Worker Process

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 3.4.1 | Chrome crashes | Chrome process dies inside worker pod. | TCP connection reset on port 9222. `TunnelForwarder` sends `{type: 'close'}`. Consumer notified. |
| 3.4.2 | Chrome hangs | Chrome stops responding but connection stays open. | Consumer-side timeout (chrome-devtools-mcp has its own timeout). Consumer may close and re-open tunnel. |
| 3.4.3 | Chrome restarts | Chrome process restarts inside worker (new PID, new browser ID). | Old tunnel connection invalid. Consumer must close and re-open tunnel. New `/json/version` returns new `webSocketDebuggerUrl`. |
| 3.4.4 | Dev server (port 3000) not started | Tunnel opened to port 3000 but dev server hasn't started yet. | TCP connection refused. Error frame returned. Consumer retries after delay. |
| 3.4.5 | Service sends unexpected data | Worker service sends malformed data. | Raw bytes forwarded as-is. Tunnel doesn't interpret content — consumer-side client handles it. |

---

## 4. Zero-Downtime Deployments

### 4.1 Relay Deployment

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 4.1.1 | Rolling update — connections migrate | Deploy relay v2 with `maxUnavailable: 0`. | New pod starts, passes readiness. Old pod gets SIGTERM, preStop sleeps 30s, then `sio.close()`. Clients auto-reconnect to new pod. |
| 4.1.2 | No request loss during rollout | Run continuous SDK requests during relay deployment. | Some requests may get 503 during reconnect window (~2-5s). SDK retries succeed. No permanent failures. |
| 4.1.3 | Active tunnels survive rollout | CDP session active during relay deployment. | Tunnel briefly disconnects (~2-5s). Socket.IO reconnects. Consumer re-sends `{type: 'open'}` for all tunnels. CDP session resumes. |
| 4.1.4 | Canary deployment | Route 10% traffic to relay v2, 90% to v1. | Both versions serve requests. Redis adapter enables cross-version routing. |

### 4.2 Mediator Deployment

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 4.2.1 | Mediator rolling update | Deploy mediator v2. | Old mediator disconnects from relay. New mediator connects. Worker TCP connections dropped — consumer re-opens tunnels. |
| 4.2.2 | Worker connections re-established | Mediator restarts with active tunnels. | All mediator → worker TCP connections lost. Consumer detects via `{type: 'close'}` frames. Consumer re-opens tunnels → mediator opens new TCP connections. |
| 4.2.3 | In-flight commands during mediator restart | Command submitted via `/control` while mediator restarting. | SDK gets timeout/error. Retries after mediator reconnects. Command eventually succeeds. |

### 4.3 Worker Deployment

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 4.3.1 | Worker pod replaced | Delete worker pod (simulating update). | Mediator's TCP connection drops. `{type: 'close'}` sent to consumer. New pod starts. Consumer re-opens tunnel to new pod. |

---

## 5. Observability

### 5.1 Metrics

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 5.1.1 | Connection metrics emitted | Mediator connects to relay. | `blitzy.tunnel.relay.active_mediators` increments. Visible in Datadog. |
| 5.1.2 | Tunnel metrics emitted | Open tunnel and send data. | `blitzy.tunnel.mediator.active_connections` increments. `blitzy.tunnel.relay.route_latency` histogram populated. |
| 5.1.3 | Error metrics emitted | Trigger connection refused on worker port. | `blitzy.tunnel.mediator.connect_errors` counter increments. |
| 5.1.4 | Disconnection metrics | Kill mediator. | `blitzy.tunnel.relay.active_mediators` decrements. `blitzy.tunnel.mediator.worker_disconnects` incremented on mediator side before shutdown. |

### 5.2 Logging

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 5.2.1 | Tunnel lifecycle logged | Open → data → close a tunnel. | Structured logs with `runner_id`, `port`, `connection_id`, `hop` tag at each component. |
| 5.2.2 | Error logged with context | Worker connection refused. | Log entry includes `runner_id`, `port`, `hop: mediator→worker`, `error: Connection refused`. |
| 5.2.3 | Logs forwarded to Datadog | Mediator emits logs via `/logs` namespace. | Logs appear in Datadog Log Explorer filtered by `company_id` and `hop` tags. |

### 5.3 Distributed Tracing

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 5.3.1 | trace_id propagated | Send CDP command through tunnel. | Same `trace_id` appears in logs at consumer, relay, mediator. Traceable end-to-end in Datadog APM. |
| 5.3.2 | Latency breakdown visible | View trace for a screenshot request. | Trace shows time spent at each hop: consumer→relay, relay→mediator, mediator→worker. |

### 5.4 Alerting

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 5.4.1 | Mediator offline alert | Disconnect mediator for > 60s. | Datadog alert fires: "Mediator disconnected for acme-corp". |
| 5.4.2 | Error spike alert | Trigger 10 connection refused errors in 1 minute. | Datadog alert fires: "Tunnel error spike for acme-corp". |
| 5.4.3 | Latency alert | Introduce 1s artificial delay at relay. | Datadog alert fires: "Tunnel latency p95 > 500ms". |

---

## 6. `/control` Namespace — SDK Operations

### 6.1 Runner Operations

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 6.1.1 | Create runner via relay | SDK calls `create_runner()`. | Request flows: SDK → `POST /relay` → relay → mediator `/control` → `RunnerService.create_runner()` → K8s deployment created. Response flows back. |
| 6.1.2 | Get runner status via relay | SDK calls `get_runner_status()`. | Routed through relay. Returns status from `JobServer` database. |
| 6.1.3 | Delete runner via relay | SDK calls `delete_runner()`. | Runner deleted. K8s deployment removed. Tunnel connections for that runner closed. |
| 6.1.4 | Submit command via relay | SDK calls `create_job_command()`. | Command queued to Redis. Execution ID returned. |
| 6.1.5 | Poll command status via relay | SDK calls `get_command_status()`. | Returns current status from database. |
| 6.1.6 | Restart session via relay | SDK calls `restart_session()`. | Restart command queued. Execution ID returned. |

### 6.2 Server Operations (Mediator → Main Server)

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 6.2.1 | Registration on connect | Mediator connects to relay. | Registration implicit — no explicit `PUT /v1/client/register` needed. Relay associates socket with company. |
| 6.2.2 | Get job info | Mediator calls `get_job_info()` via WS. | Request flows: mediator → relay → main server. Response returned. |
| 6.2.3 | Register job | Mediator calls `register_job()` via WS. | Request flows through relay to main server. Job registered. |
| 6.2.4 | Secret sync | Mediator calls `request_secret_sync()` via WS. | Sync request forwarded through relay. |

---

## 7. `/logs` Namespace — Log Forwarding

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 7.1 | Log batch emitted | Mediator emits `log_batch` event with 50 log entries. | Relay receives on `/logs` namespace. |
| 7.2 | Relay batches and forwards | Relay accumulates logs for 5s or 100 entries. | Batch forwarded to Datadog Logs Intake API. |
| 7.3 | Logs appear in Datadog | Trigger known log message from mediator. | Log visible in Datadog Log Explorer with correct `company_id`, `source`, `pod_name` tags. |
| 7.4 | Log forwarding under backpressure | Datadog Intake API slow (simulated 5s latency). | Relay drops oldest logs rather than blocking the WebSocket. No impact on `/control` or `/tunnel`. |
| 7.5 | Large log volume | Mediator emits 1000 log entries/s. | Relay samples or batches. No memory leak. Datadog receives representative sample. |

---

## 8. `/metrics` Namespace — Metrics Forwarding

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 8.1 | Metric batch emitted | Mediator emits `metric_batch` with CPU, memory, queue depth. | Relay receives on `/metrics` namespace. |
| 8.2 | Relay forwards to Datadog | Relay forwards to Datadog Metrics Intake API. | Metrics visible in Datadog Metrics Explorer. |
| 8.3 | Custom metrics | Emit `blitzy.runner.cpu_usage` with `runner_id` tag. | Visible in Datadog with correct tags. Can build dashboards per customer, per runner. |
| 8.4 | Metrics forwarding failure | Datadog API key invalid. | Relay logs error. Metrics dropped. No impact on other namespaces. |

---

## 9. Security

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 9.1 | Unauthenticated WebSocket connection | Connect to relay without `api_key`. | Connection rejected. |
| 9.2 | Unauthenticated HTTP request | `POST /relay` without `Authorization` header. | HTTP 401 Unauthorized. |
| 9.3 | Cross-tenant tunnel isolation | Company A tries to send tunnel frame with Company B's `runner_id`. | Mediator rejects — `runner_id` not owned by this company. Error returned. |
| 9.4 | Tunnel to unauthorized port | Open tunnel to port 22 (SSH) on worker. | Mediator rejects — only allowed ports (configurable allowlist). |
| 9.5 | Replay attack | Replay a captured tunnel frame. | `connection_id` no longer valid. Mediator rejects. |
| 9.6 | Data in transit encryption | Inspect traffic between consumer and relay. | All traffic over WSS (TLS). No plaintext. |
| 9.7 | No Chrome exposed publicly | Port scan customer cluster from outside. | Port 9222 not reachable from outside the cluster. Only accessible via K8s internal service. |

---

## 10. Performance

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 10.1 | Tunnel latency baseline | Measure round-trip for 1 KB CDP command through all hops. | < 50ms total (excluding Chrome execution time). |
| 10.2 | Screenshot latency | Capture full-page screenshot through tunnel. | < 500ms overhead from tunnel (Chrome execution time excluded). |
| 10.3 | Concurrent tunnels | Open 10 tunnels to different ports on same runner. | All functional. No interference. Memory < 50 MB on consumer. |
| 10.4 | Concurrent runners with tunnels | 5 runners each with 2 tunnels. | 10 tunnels active. Mediator maintains 10 TCP connections. No resource exhaustion. |
| 10.5 | Sustained throughput | Stream 100 KB/s through tunnel for 1 hour. | No memory leak. Latency stable. No connection drops. |
| 10.6 | Idle tunnel keepalive | Open tunnel, no traffic for 30 minutes. | Tunnel stays alive (Socket.IO heartbeat keeps connection open). Next frame flows immediately. |

---

## 11. Local Development (Mode A — Telepresence)

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 11.1 | No relay configured | Start mediator without `WS_RELAY_URL`. | Flask HTTP routes active. No Socket.IO client started. All existing functionality works. |
| 11.2 | SDK connects directly | SDK calls mediator HTTP endpoints directly (no relay). | Runner CRUD, command submission work as today. |
| 11.3 | Local Chrome | chrome-devtools-mcp connects to local Chrome. | No tunnel needed. Direct connection to `localhost:9222` on developer's machine. |

---

## 12. Integration Testing (Mode B — docker-compose)

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 12.1 | Full stack startup | `docker-compose up` with relay, mediator, worker, Redis. | All services healthy. Mediator connected to relay. |
| 12.2 | End-to-end runner lifecycle | Create runner → submit command → poll status → delete runner. | All operations succeed through relay → mediator → Redis → worker path. |
| 12.3 | End-to-end tunnel | Open tunnel to Chrome → take screenshot → close tunnel. | Screenshot data flows through all hops. Image valid. |
| 12.4 | End-to-end log forwarding | Mediator emits logs → relay receives → mock Datadog endpoint receives. | Logs arrive with correct tags and structure. |
