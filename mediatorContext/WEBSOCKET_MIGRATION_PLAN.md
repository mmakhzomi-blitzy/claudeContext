# WebSocket Migration Plan

## Context

The mediator currently exposes a public HTTP API via Envoy Gateway. Consumer jobs (Cloud Run) call the mediator directly via BlitzyClient SDK. Chrome CDP routes are also exposed publicly via Envoy with zero authentication — a security risk for enterprise client data.

**Goal**: Replace SDK and CDP communication with a single outbound WebSocket connection from the mediator to a new WS Relay Service on Blitzy K8s. Chrome CDP is tunneled through the WebSocket, fully authenticated. Envoy Gateway stays as-is (serves its own purpose). Internal communication (mediator ↔ worker via Redis) stays unchanged.

---

## Architecture

```
══════════════════════════════════════════════════════════════════════════════
                         BLITZY CLOUD (GKE)
══════════════════════════════════════════════════════════════════════════════

  Consumer Job (Cloud Run)                Admin Service
  ┌──────────────────────────────────┐    ┌──────────────────────────┐
  │                                  │    │                          │
  │  BlitzyClient SDK ──HTTP──┐      │    │  BlitzyClient            │
  │  (X-Client-ID header)     │      │    │  (secret push via relay) │
  │                           │      │    │                          │
  │  chrome-devtools-mcp      │      │    └──────────┬───────────────┘
  │    → Local CDP Proxy      │      │               │ internal K8s svc
  │      (localhost:9222) ────┤      │               │
  └───────────────────────────┼──────┘               │
                              │                      │
                              ▼                      ▼
  GKE Gateway (os.api-k.blitzy.dev)
  ┌─────────────────────────────────────────────────────────────────────┐
  │  /v1/relay/*  → Relay Service     (path-prefix routing)            │
  │  /v1/client/* → Admin Service     (same pattern as /v1/api/chat)   │
  └──────────────────────┬──────────────────────────────────────────────┘
                         │
  WS Relay Service       ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │                                                                     │
  │  Transparent HTTP proxy          Socket.IO Server                   │
  │  (any path + X-Client-ID)       • /control namespace (runner/cmd)  │
  │  GET /health/connections         • /cdp namespace (Chrome tunnel)   │
  │                                                                     │
  │  Redis (connection registry, Socket.IO adapter for multi-replica)   │
  └──────────────────────────────────┬──────────────────────────────────┘
                                     │
          Single persistent WSS      │  Authenticated with API_KEY
          (Socket.IO, multiplexed)   │  Via MAIN_SERVER_URL/v1/relay
                                     │
══════════════════════════════════════╪═══════════════════════════════════
                PUBLIC INTERNET      │
══════════════════════════════════════╪═══════════════════════════════════
                                     │
  CUSTOMER KUBERNETES CLUSTER        │
  ┌──────────────────────────────────┼──────────────────────────────────┐
  │                                  │                                  │
  │  Mediator Pod                    │                                  │
  │  ┌───────────────────────────────▼────────────────────────────┐     │
  │  │                                                            │     │
  │  │  Socket.IO Client                                          │     │
  │  │    connects to MAIN_SERVER_URL/v1/relay/socket.io          │     │
  │  │    /control → Message Router → RunnerService,              │     │
  │  │                                 CommandService, etc.       │     │
  │  │    /cdp     → CDP Forwarder → Worker Chrome :9222          │     │
  │  │                                                            │     │
  │  │  Flask (:8080) — health probes only (K8s internal)         │     │
  │  │                                                            │     │
  │  └───────────┬──────────────────────────┬─────────────────────┘     │
  │              │ Redis                     │ K8s internal network      │
  │              │                           │                          │
  │  ┌───────────▼──────────┐   ┌───────────▼──────────┐               │
  │  │ Worker Pod (Job A)   │   │ Worker Pod (Job B)   │   ...         │
  │  │ • Bash Session       │   │ • Bash Session       │               │
  │  │ • Chrome :9222       │   │ • Chrome :9222       │               │
  │  │ • Dev server :3000   │   │ • Dev server :3000   │               │
  │  └─────────────────────┘   └─────────────────────┘               │
  │                                                                     │
  └─────────────────────────────────────────────────────────────────────┘
```

---

## Components

### 1. WS Relay Service (NEW — Blitzy K8s)

A small Python service: Socket.IO server + transparent HTTP proxy. Bridges HTTP from callers (SDK, admin service) to WebSocket to mediator. Deployed behind the existing GKE Gateway at `os.api-k.blitzy.dev` under the `/v1/relay` path prefix.

**Libraries**: `flask-socketio` + `gevent` (consistent with existing Flask tooling)

**Responsibilities**:
- Accept persistent WebSocket connections from mediators (Socket.IO server)
- Authenticate mediators via `API_KEY` on connect
- Maintain connection registry in Redis (client_id → socket_id)
- Transparent HTTP proxy — any HTTP path with `X-Client-ID` header gets wrapped and forwarded to the correct mediator via WS
- Expose health API (`GET /health/connections`) — list connected mediators and their status
- Tunnel CDP frames between consumer's local proxy and mediator (`/cdp` namespace)
- Multi-replica via Socket.IO Redis adapter

**Routing**: Callers send normal HTTP to `os.api-k.blitzy.dev/v1/relay/...`. GKE Gateway strips `/v1/relay` prefix and routes to relay service. Relay sees the original path (e.g., `/api/v1/runners`), wraps it, and sends to mediator via WS.

**Estimated size**: ~500-700 lines

```python
# Core relay logic
from flask import Flask, request, jsonify
from flask_socketio import SocketIO, Namespace, join_room

app = Flask(__name__)
redis_url = f"redis://{REDIS_USERNAME}:{REDIS_PASSWORD}@{REDIS_HOST}:{REDIS_PORT}"
socketio = SocketIO(app, message_queue=redis_url, async_mode='gevent')

# Transparent HTTP proxy — any path with X-Client-ID
@app.route('/', defaults={'path': ''}, methods=['GET', 'POST', 'PUT', 'DELETE'])
@app.route('/<path:path>', methods=['GET', 'POST', 'PUT', 'DELETE'])
def relay_request(path):
    client_id = request.headers.get('X-Client-ID')
    if not client_id:
        return jsonify({"error": "Missing X-Client-ID"}), 400
    if not registry.is_online(client_id):
        return jsonify({"error": "Mediator offline"}), 503
    payload = {"method": request.method, "path": f"/{path}",
               "body": request.get_json(silent=True), "query": dict(request.args)}
    response = socketio.call('request', payload, room=client_id,
                             namespace='/control', timeout=30)
    return jsonify(response)

# Socket.IO: mediator registration
class ControlNamespace(Namespace):
    def on_register(self, data):
        client_id = authenticate(data['api_key'])
        join_room(client_id)
        registry.register(client_id, request.sid)
```

### 2. Mediator — Socket.IO Client (NEW module)

**File**: `src/services/ws_client.py` (new)

Connects outbound to the relay. Registers with `API_KEY` — the relay resolves this to the mediator's `client_id` (`ClientInstallation.id`) and places the socket in the corresponding room. Receives relayed requests on `/control` namespace, CDP frames on `/cdp` namespace.

```python
sio = socketio.AsyncClient()
# Connect to relay via MAIN_SERVER_URL (already in config) + /v1/relay path
relay_ws_url = f"{MAIN_SERVER_URL}/v1/relay"
await sio.connect(relay_ws_url, auth={'api_key': API_KEY}, namespaces=['/control', '/cdp'])

@sio.on('request', namespace='/control')
async def handle_request(data):
    result = await router.dispatch(data)
    return result  # Socket.IO ack sends response back
```

**Conditional activation**: Connects when `MAIN_SERVER_URL` is set and relay is reachable. Otherwise Flask HTTP routes remain active (local dev mode). No separate `WS_RELAY_URL` env var needed — the relay is at `MAIN_SERVER_URL/v1/relay`.

### 3. Mediator — Message Router (NEW module)

**File**: `src/services/ws_router.py` (new)

Dispatches incoming WS messages to existing service layer. Maps `path` + `method` to existing service calls. The service layer (RunnerService, CommandService, SecretsService) stays **unchanged**.

### 4. Mediator — CDP Forwarder (NEW module)

**File**: `src/services/cdp_forwarder.py` (new)

Receives CDP frames from `/cdp` namespace, forwards to the correct worker's Chrome via internal K8s network.

**Responsibilities**:
- Maintain a pool of WebSocket connections to worker Chrome instances (one per active runner with Chrome capability)
- Open connection to worker Chrome on first CDP request for a runner
- Close connection when runner is deleted
- Forward CDP frames bidirectionally (relay ↔ worker Chrome)

**Library**: `websockets` (lightweight, raw WS for the Chrome connection)

**Estimated size**: ~150-200 lines

### 5. Consumer — Local CDP Proxy (NEW module)

**File**: `blitzy-utils-python/blitzy_utils/blitzy_utils/runner_ops/cdp_proxy.py` (new)

Small local proxy on the consumer side. `chrome-devtools-mcp` connects to `localhost:9222`, proxy tunnels frames through the relay to the mediator.

**Handles both CDP protocols**:
- HTTP discovery (`GET /json/version`, `GET /json/list`) — tunnels to real Chrome via relay, returns response as-is (URLs already say `localhost:9222`, which matches the proxy)
- WebSocket control (`/devtools/browser/...`) — tunnels frames bidirectionally through relay

**Library**: `websockets`

**Estimated size**: see implementation details below

**Consumer usage** (helper.py):
```python
if self.runner_session and self.runner_session.chrome_url:
    # Start local CDP proxy that tunnels through relay
    cdp_proxy = CDPProxy(relay_url=WS_RELAY_URL, runner_id=runner_id)
    cdp_proxy.start()  # listens on localhost:9222
    chrome_config = {
        "transport": "stdio", "command": "npx",
        "args": ["chrome-devtools-mcp@latest", "--browserUrl=http://localhost:9222"],
    }
else:
    chrome_config = CHROME_DEVTOOLS_MCP  # local Chrome (existing behavior)
```

### 6. SDK (BlitzyClient) — Unified Relay Communication

**File**: `blitzy-utils-python/blitzy_utils/blitzy_utils/blitzy_client.py`

`BlitzyClient` becomes the **single SDK for all relay-bound communication**. Any service that needs to talk to a mediator through the relay uses this SDK — no separate HTTP clients.

**Callers after migration:**

| Caller | Usage |
|--------|-------|
| Consumer jobs | `BlitzyClient.create_runner()`, `submit_command()`, etc. — same as today |
| Admin service (secret push) | `BlitzyClient` replaces internal `HttpForwardingClient` — secrets forwarded through relay via the same SDK |
| Any future Blitzy service | Import `BlitzyClient`, set `client_id`, call through relay |

Two changes to the SDK:
1. **Add `X-Client-ID` header** to all HTTP requests — the `ClientInstallation.id` returned by the admin service
2. **URL resolution** — `get_client_url()` returns the relay URL (same for all clients) + caches the `client_id` for routing

The admin service (`archie-service-admin`) already resolves "which ClientInstallation serves this entity" via the `GET /v1/client/entity/{type}/{id}` endpoint. Today it returns `configuration.url` (the mediator's direct URL). After migration it also returns `configuration.client_id` (the `ClientInstallation.id`):

```python
# Current response from admin service
{"configuration": {"url": "https://acme-client.blitzy.dev"}}

# After migration
{"configuration": {"url": "https://relay.blitzy.internal", "client_id": "inst_abc123"}}
```

The SDK caches both values and includes `X-Client-ID` on every request:

```python
def get_client_url(self, company_id: str) -> str:
    if self._server_url:
        return self._server_url
    with self._service_client_factory() as client:
        response = client.get("admin", f"/v1/client/entity/COMPANY/{company_id}")
        configuration = response.json().get("configuration", {})
        self._server_url = configuration.get("url")
        self._client_id = configuration.get("client_id")  # NEW — routing key
    return self._server_url
```

**Default installation handling**: When no dedicated `ClientInstallation` exists for an entity, the admin service returns the `BLITZY_SHARED` default installation's `client_id`. The SDK and relay don't know or care — it's just an ID. The "which client to use" decision is fully centralized in the admin service.

**Lookup key flexibility**: Today the SDK calls `get_client_url(company_id)`. If the lookup key changes in the future (e.g., `team_id`, `user_id`), that change is isolated to the SDK and admin service. The relay only ever sees `client_id` — it never touches the lookup key.

Everything else stays identical — same methods, same request/response shapes, same polling logic.

### SDK implementation diff

The SDK has two internal HTTP clients with different responsibilities:

```
BlitzyClient
  ├── ServiceClient (via _service_client_factory)
  │     Used ONLY in get_client_url() to call admin API
  │     Authenticated with Google IAM tokens, uses httpx
  │
  └── _RequestsAdapter (via _http_client)
        Used by ALL other methods (create_runner, get_command_status, etc.)
        Plain HTTP via requests library, no auth
        Injected via BlitzyHttpClient Protocol for testability
```

The `BlitzyHttpClient` Protocol enables dependency injection — tests pass a mock `http_client`:

```python
class BlitzyHttpClient(Protocol):
    def get(self, url: str, **kwargs: Any) -> Any: ...
    def post(self, url: str, **kwargs: Any) -> Any: ...
    def delete(self, url: str, **kwargs: Any) -> Any: ...
```

Every SDK method follows the same 3-step pattern:

```python
def any_method(self, company_id, ...):
    server_url = self.get_client_url(company_id)    # Step 1: resolve URL
    url = f"{server_url}{api_path}"                  # Step 2: build full URL
    response = self._http_client.post(url, ...)      # Step 3: HTTP call
    return response.json()
```

**The complete diff (zero breaking changes):**

```diff
  class _RequestsAdapter:
+     def __init__(self):
+         self._client_id: str | None = None
+
+     def _with_client_id(self, kwargs: dict) -> dict:
+         """Inject X-Client-ID header when client_id is available."""
+         if self._client_id:
+             headers = kwargs.get("headers") or {}
+             headers["X-Client-ID"] = self._client_id
+             kwargs["headers"] = headers
+         return kwargs
+
      def get(self, url: str, **kwargs: Any) -> Any:
-         return requests.get(url, **kwargs)
+         return requests.get(url, **self._with_client_id(kwargs))
      def post(self, url: str, **kwargs: Any) -> Any:
-         return requests.post(url, **kwargs)
+         return requests.post(url, **self._with_client_id(kwargs))
      def delete(self, url: str, **kwargs: Any) -> Any:
-         return requests.delete(url, **kwargs)
+         return requests.delete(url, **self._with_client_id(kwargs))


  class BlitzyClient:
      def __init__(self, ...):
          ...
          self._server_url: Optional[str] = None
+         self._client_id: Optional[str] = None
          self._company_id: Optional[str] = None

      def get_client_url(self, company_id: str) -> str:
          ...
              self._server_url = server_url.rstrip("/")
+             self._client_id = configuration.get("client_id")
+             if self._client_id and isinstance(self._http_client, _RequestsAdapter):
+                 self._http_client._client_id = self._client_id
              self._company_id = company_id
              return self._server_url
```

**Backwards compatible**: When `client_id` is `None` (admin service not yet updated, or local dev without relay), no header is injected. The SDK works exactly as before.

**What does NOT change:**

| Component | Changes? | Why |
|---|---|---|
| `BlitzyHttpClient` Protocol | No | `_with_client_id` modifies kwargs before same `.get/.post/.delete` |
| Method signatures (`create_runner`, `delete_runner`, etc.) | No | Same params, same return types |
| Request/response JSON payloads | No | Same bodies, same response shapes |
| `RunnerSession` | No | Calls BlitzyClient methods — transparent |
| Module-level convenience wrappers | No | Delegate to `_get_default_client()` |
| `wait_for_ready` / polling logic | No | Calls `get_runner_status` in a loop — same |
| `execute_command` / `execute_bash_tool` | No | Calls `create_job_command` + `get_command_status` — same |
| `ServiceClient` | No | Still used in `get_client_url` to call admin API |

### Data model reference

The routing resolution leverages existing tables in `db-common-model`:

```
ClientInstallation
  ├── id (PK) ← this is the client_id / routing key for the relay
  ├── company_id (FK)
  ├── client_type: BLITZY_SHARED | CUSTOMER_DEDICATED
  ├── is_default: bool
  ├── configuration: JSONB (contains url, client_id after migration)
  └── status: ACTIVE | SUSPENDED | DECOMMISSIONED

ClientInstallationAccess
  ├── client_id (FK → ClientInstallation.id)
  ├── entity_type: USER | TEAM | COMPANY
  ├── entity_id (user_id, team_id, or company_id)
  └── is_default: bool (default client for this entity)

Subscription
  ├── plan_name: FREE | PRO | TEAMS | ENTERPRISE
  └── plan_owner_id → User (FREE/PRO) | Team (TEAMS) | Company (ENTERPRISE)
```

Resolution chain: entity → `ClientInstallationAccess` → `ClientInstallation.id` → relay routes by this ID.
For entities without a dedicated installation, admin service falls back to the `BLITZY_SHARED` default.

### 7. Mediator — BlitzyService Changes

**File**: `src/services/blitzy_service.py`

Outbound calls from mediator to admin service (`get_job_info`, `register_job`, `request_secret_sync`) **stay as direct HTTP** via `HttpClient`. The mediator can still make outbound HTTP calls — only inbound is removed. No transport change needed for these.

**Registration change**: The `register()` method currently sends `{url: BLITZY_CLIENT_URL}` via `PUT /v1/client/register`. After migration the mediator has no public URL to advertise. The registration call becomes a heartbeat/presence signal only — it no longer sends a URL. Admin reaches the mediator through the relay (internal K8s service) using `BlitzyClient` + `client_id`.

### 8. Mediator — Flask & main.py

**File**: `main.py`

- Flask stays for K8s health probes (`/api/v1/health-check`, `/api/v1/uptime-check`, `/api/v1/redis-check`)
- All other Flask routes stay in codebase but are only active in local dev mode
- When `MAIN_SERVER_URL` is set: start Socket.IO client alongside RQ worker, connecting to `MAIN_SERVER_URL/v1/relay`

```python
if MAIN_SERVER_URL:
    relay_url = f"{MAIN_SERVER_URL}/v1/relay"
    start_ws_client(relay_url)
else:
    logger.info("No MAIN_SERVER_URL — running in HTTP-only mode (local dev)")
```

### 9. Authentication

Same `API_KEY` env var. Sent during the WebSocket connection handshake as an HTTP header in the upgrade request (`Authorization: Bearer <API_KEY>`). No new auth mechanism.

The relay validates the API key on connect, resolves it to a `client_id` (`ClientInstallation.id`), and builds the connection registry.

SDK requests include `X-Client-ID` header for routing + existing `Authorization` header for authentication.

### 10. Code Removal — CDP Route Creation

With CDP tunneled through WebSocket, the dynamic K8s CDP routing code is removed. **Envoy Gateway stays as-is** — it serves its own purpose.

**Mediator CDP code removed**:
- `create_chrome_route()` in `KubernetesService`
- `delete_chrome_route()` in `KubernetesService`
- `_build_chrome_service_name()`, `_build_chrome_route_name()`
- Chrome route creation/cleanup in `RunnerService.create_runner()`, `delete_runner()`, `_cleanup_failed_creation()`
- `capabilities_metadata.chrome_url` as a public URL (becomes tunnel identifier)

**Envoy Gateway — NO changes** (`archie-helm-chart/blitzy-client/chart/`):
- Gateway stays enabled, all templates stay
- `BLITZY_CLIENT_URL`, `GATEWAY_HOST`, `GATEWAY_NAME` stay in config
- cert-manager, gateway-helm dependencies stay

**What stays**:
- Chrome in worker Dockerfile (still runs, just not exposed via gateway)
- `RUNNER_CAPABILITIES` env var injection
- `StartChromeStep` in worker startup
- Capability framework (RunnerCapability enum, capabilities in RunnerConfig)
- `CustomObjectsApi` in kube client provider (gateway may use it)

---

## Namespace Design

All namespaces share a single WebSocket connection per mediator. Socket.IO has no limit on namespaces or rooms ([confirmed by maintainer](https://github.com/socketio/socket.io/discussions/4754)). Rooms are lightweight in-memory Sets (~100 bytes each).

### Namespaces

| Namespace | Purpose | Direction | Traffic pattern | Phase |
|-----------|---------|-----------|-----------------|-------|
| `/control` | Runner CRUD, command submit/status, server requests (SDK operations) | Bidirectional | Low volume, latency-sensitive. Typical 1-50 KB per message. | Phase 1 |
| `/cdp` | Chrome DevTools Protocol tunnel (screenshots, navigation, evaluation) | Bidirectional | Bursty, large payloads. Screenshots up to 5 MB. | Phase 3 |
| `/logs` | Cluster log forwarding → Datadog | Mediator → Relay | Continuous stream. Typical 1-50 KB/s per cluster. | Future |
| `/metrics` | Cluster metrics forwarding → Datadog | Mediator → Relay | Periodic. Typical 1-10 KB per batch, every 10-60s. | Future |

### Room structure (within each namespace)

Rooms are keyed by `client_id` (`ClientInstallation.id`). The relay has no concept of company, team, or user — only `client_id`.

```
/control
  ├── room: "inst_abc123"        ← routes SDK requests to this mediator
  └── room: "inst_shared01"      ← default (BLITZY_SHARED) mediator

/cdp
  ├── room: "runner-abc123"      ← routes CDP frames to correct runner's Chrome
  └── room: "runner-def456"

/logs
  ├── room: "inst_abc123"        ← all logs from this client installation
  └── room: "inst_shared01"

/metrics
  ├── room: "inst_abc123"        ← all metrics from this client installation
  └── room: "inst_shared01"
```

### `/control` — SDK Operations

The core request/response namespace. Handles all runner and command operations that the BlitzyClient SDK currently performs via HTTP.

**Events:**
- `request` — relay forwards SDK HTTP request to mediator, waits for ack response
- `server_request` — mediator sends outbound request to main server (register_job, get_job_info, etc.)

### `/cdp` — Chrome DevTools Protocol Tunnel

Tunnels CDP frames between the consumer's local proxy (localhost:9222) and the worker's Chrome instance via the mediator's CDP forwarder.

**Events:**
- `cdp_http` — HTTP discovery requests (`/json/version`, `/json/list`)
- `cdp_ws_open` — open a CDP WebSocket channel to a specific runner's Chrome
- `cdp_ws_frame` — bidirectional CDP WebSocket frame
- `cdp_ws_close` — close a CDP WebSocket channel

### `/logs` — Log Forwarding to Datadog

Streams cluster logs from customer K8s through the relay to Datadog. The mediator collects logs from pods (mediator + workers) and forwards them over the existing WebSocket — no separate outbound connection needed from the customer cluster.

**Flow:**
```
Customer K8s                    Blitzy Cloud
┌──────────────┐               ┌──────────────┐               ┌──────────┐
│ Mediator     │   /logs       │ Relay        │   HTTP POST   │ Datadog  │
│              ├───────────────►              ├───────────────► Logs     │
│ • Pod logs   │               │ • Buffer     │               │ Intake   │
│ • K8s events │               │ • Batch      │               │ API      │
│ • App logs   │               │ • Forward    │               │          │
└──────────────┘               └──────────────┘               └──────────┘
```

**Events:**
- `log_batch` — mediator sends a batch of log entries (JSON array)

**Log entry schema:**
```json
{
  "timestamp": "2026-03-17T10:30:00.000Z",
  "source": "mediator|worker|k8s",
  "pod_name": "worker-abc123",
  "level": "INFO",
  "message": "Command executed successfully",
  "tags": {"company_id": "acme-corp", "runner_id": "abc123", "job_id": "job-456"}
}
```

**Relay-side forwarding:**
- Batches incoming logs (buffer for 5-10s or 100 entries, whichever comes first)
- Forwards to Datadog via [Logs HTTP Intake API](https://docs.datadoghq.com/api/latest/logs/) (`POST https://http-intake.logs.datadoghq.com/api/v2/logs`)
- Tags with `company_id`, `cluster_id` for filtering in Datadog
- Drops logs on backpressure rather than blocking the WebSocket

**Volume estimates:**

| Log source | Volume per cluster | Notes |
|------------|-------------------|-------|
| Mediator pod stdout/stderr | 1-5 KB/s | Application logs |
| Worker pods (per runner) | 1-10 KB/s | Build output, command execution |
| K8s events | Bursty, ~1 KB/event | Deployments, errors, OOMKills |
| **Total per cluster** | **5-50 KB/s typical** | Well within socket capacity |

### `/metrics` — Metrics Forwarding to Datadog

Streams cluster-level metrics from customer K8s through the relay to Datadog. Provides observability into remote clusters without requiring direct Datadog agent access from the customer environment.

**Flow:**
```
Customer K8s                    Blitzy Cloud
┌──────────────┐               ┌──────────────┐               ┌──────────┐
│ Mediator     │   /metrics    │ Relay        │   HTTP POST   │ Datadog  │
│              ├───────────────►              ├───────────────► Metrics  │
│ • Pod CPU/   │               │ • Aggregate  │               │ Intake   │
│   Memory     │               │ • Forward    │               │ API      │
│ • Runner     │               │              │               │          │
│   health     │               │              │               │          │
│ • Queue      │               │              │               │          │
│   depth      │               │              │               │          │
└──────────────┘               └──────────────┘               └──────────┘
```

**Events:**
- `metric_batch` — mediator sends a batch of metric data points

**Metric entry schema:**
```json
{
  "metric": "blitzy.runner.cpu_usage",
  "type": "gauge",
  "timestamp": 1710672600,
  "value": 72.5,
  "tags": ["company_id:acme-corp", "runner_id:abc123", "pod:worker-abc123"]
}
```

**Candidate metrics:**

| Metric | Type | Collection interval |
|--------|------|-------------------|
| `blitzy.runner.cpu_usage` | gauge | 30s |
| `blitzy.runner.memory_usage` | gauge | 30s |
| `blitzy.runner.status` | gauge (enum) | 30s |
| `blitzy.command.queue_depth` | gauge | 10s |
| `blitzy.command.execution_time` | histogram | per command |
| `blitzy.mediator.ws_connected` | gauge | 30s |
| `blitzy.mediator.uptime` | gauge | 60s |

**Relay-side forwarding:**
- Forwards to Datadog via [Metrics HTTP Intake API](https://docs.datadoghq.com/api/latest/metrics/) (`POST https://api.datadoghq.com/api/v2/series`)
- Aggregates metrics per customer before forwarding (reduces API calls)
- Uses `DD_API_KEY` env var on the relay (single Datadog API key for all customers)

**Volume estimates:**

| Source | Volume per cluster | Notes |
|--------|-------------------|-------|
| Pod metrics (7 metrics × 5 pods) | ~3 KB per batch | Every 30s |
| Command metrics | ~500 bytes per command | Per execution |
| **Total per cluster** | **~1-5 KB/s** | Negligible |

### Namespace isolation benefits

Each namespace has independent:
- **Event handlers** — `/logs` handlers don't interfere with `/control`
- **Middleware** — `/logs` can have rate limiting, `/cdp` can have larger buffer sizes
- **Error boundaries** — a crash in log processing doesn't affect command operations
- **Future scaling** — if `/logs` becomes too heavy, it can be split to a separate connection without changing `/control` or `/cdp`

---

## Connection Health Monitoring

**Socket.IO built-in**: Heartbeat ping/pong (~25s interval). `connect`/`disconnect` events fire automatically.

**Connection registry** (Redis — required for multi-replica relay):
```python
# Key: mediator:{client_id}   (ClientInstallation.id)
# Value: {sid, status, replica_id, connected_at, last_heartbeat}
```

**Health API** on relay:
```
GET /health/connections
→ {"inst_abc123": {"status": "alive", "uptime": "4h32m"}, "inst_shared01": {"status": "alive", "uptime": "12h5m"}, ...}
```

**When mediator is offline**: Relay returns HTTP 503 to SDK. SDK's existing retry/error handling surfaces it.

---

## Multi-Replica Request Routing

When the relay scales to multiple replicas, an SDK HTTP request can land on **any** pod (K8s Service round-robin). The mediator's WebSocket lives on a **specific** pod. The Socket.IO Redis adapter solves this via **pub/sub** — not queuing.

### How it works

```
SDK HTTP request (POST /relay, X-Client-ID: inst_abc123)
       │
       ▼  (K8s Service round-robin)
  Relay Pod A  (mediator NOT connected here)
       │
       │  sio.call('request', data, room='inst_abc123')
       │         │
       │         ▼  Redis PUBLISH to channel "room:inst_abc123"
       │
  Relay Pod B  (mediator WS lives here, subscribed to "room:inst_abc123")
       │
       │  delivers message to local WebSocket
       ▼
  Mediator processes request, returns via Socket.IO ack
       │
       ▼  ack flows back through Redis pub/sub to Pod A
       │
  Pod A returns HTTP response to SDK
```

### Key points

- **No sticky sessions at the HTTP layer** — any pod handles any HTTP request
- **Redis pub/sub, not a queue** — `PUBLISH`/`SUBSCRIBE`, not `RPUSH`/`BLPOP`. Real-time broadcast, ~0.1-0.5ms latency
- **If nobody is subscribed** (mediator offline) — the publish goes nowhere, `sio.call()` times out, relay returns 503
- **Works for all namespaces** — `/control` requests, `/tunnel` frames, `/logs`, `/metrics` all route through the same Redis adapter

```python
# Relay — any pod handles any HTTP request
async def relay_request(request):
    client_id = request.headers["X-Client-ID"]  # ClientInstallation.id

    # sio.call() transparently routes across replicas via Redis pub/sub
    # No pod discovery, no sticky sessions, no connection lookup
    response = await sio.call(
        'request',
        await request.json(),
        room=client_id,
        namespace='/control',
        timeout=30
    )
    return web.json_response(response)
```

### Why pub/sub, not a queue

| Aspect | Redis Pub/Sub (what we use) | Redis Queue (mediator ↔ worker) |
|--------|----------------------------|--------------------------------|
| Pattern | Broadcast to subscribers | RPUSH/BLPOP |
| Delivery | Real-time, fire-and-forget | Guaranteed, persisted |
| If nobody listening | Message dropped | Message waits in queue |
| Latency | ~0.1-0.5ms | ~1-5ms |
| Use case | Cross-replica socket routing | Async command execution |

---

## Rolling Updates & Zero-Downtime Deployments

### Relay rolling update sequence

When deploying a new relay version, K8s terminates pods one at a time. Mediators connected to a dying pod must reconnect to a surviving pod.

```
Timeline (2 replicas: Pod A, Pod B → Pod A, Pod C):
────────────────────────────────────────────────────────────────────
t=0    K8s starts new Pod C (maxSurge: 1)
t=10   Pod C passes readiness probe, enters Service endpoints
t=11   K8s marks Pod B for termination
t=11   Pod B removed from Service endpoints (no new HTTP/WS traffic)
t=11   preStop hook fires → sleep 30s (drain period)
t=11   Mediator on Pod B detects disconnect (Socket.IO 'disconnect' event)
t=12   Mediator auto-reconnects → lands on Pod A or Pod C
t=12   Mediator re-registers in room → Redis registry updated
t=13   Mediator fully operational on new pod (~1-2s total downtime)
  ...
t=41   preStop sleep ends, Pod B receives SIGTERM
t=41   Pod B shuts down gracefully (already drained)
────────────────────────────────────────────────────────────────────
```

### K8s deployment spec

```yaml
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0    # never kill a pod before new one is ready
      maxSurge: 1          # one extra pod during rollout
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - lifecycle:
            preStop:
              exec:
                command: ["python", "-c", "import time; time.sleep(30)"]
```

### Mediator reconnection protocol

The mediator treats every `connect` event as a fresh registration. No "resume session" complexity.

```python
@sio.event(namespace='/control')
async def connect():
    # Re-register on every connect (including reconnect after relay update)
    # API_KEY resolves to client_id (ClientInstallation.id) on the relay side
    await sio.emit('register', {
        'api_key': API_KEY,
    }, namespace='/control')
    logger.info("registered_with_relay")

# Socket.IO client config — infinite auto-reconnect
await sio.connect(
    WS_RELAY_URL,
    namespaces=['/control', '/tunnel', '/logs', '/metrics'],
    reconnection=True,           # auto-reconnect (default)
    reconnection_attempts=0,     # infinite retries
    reconnection_delay=1,        # start at 1s
    reconnection_delay_max=5,    # cap at 5s
)
```

### In-flight request handling during reconnect

| Scenario | What happens | Recovery |
|---|---|---|
| HTTP request arrives, mediator offline (reconnecting) | `sio.call()` times out → relay returns 503 | SDK retries |
| HTTP request in progress, relay pod dies mid-response | `sio.call()` times out → relay returns 504 | SDK retries |
| Tunnel frame in transit | Frame lost during ~1-2s reconnect window | Chrome/client retries at protocol level |
| Mediator → relay outbound request (server_request) | Mediator buffers or retries after reconnect | Socket.IO client queues events during reconnect |

**Why ~1-2s downtime is acceptable**: LLM thinking time between SDK calls is 5-30+ seconds. The reconnect blip is invisible to the consumer job.

---

## Industry Patterns for Multi-Node WebSocket Routing

Cross-replica WebSocket routing is a well-solved problem. Four patterns exist in the industry, each suited to a different scale.

### Pattern 1: Pub/Sub Backplane

Every server subscribes to a shared message bus (Redis, Kafka, NATS). When a message needs to reach a specific client, it's published to the bus. The server holding that client delivers it locally. No server-to-server knowledge needed.

> "Add a pub/sub backplane (Redis, Kafka, NATS, or a message broker) between your WebSocket servers. Each server maintains local connections and publishes messages to a shared broker, which distributes them across the cluster."
> — [WebSocket.org: WebSockets at Scale](https://websocket.org/guides/websockets-at-scale/)

> "A significant advantage of adopting the pub/sub pattern is that you often have only one component that has to deal with scaling WebSocket connections — the message broker."
> — [Ably: Scaling Pub/Sub with WebSockets and Redis](https://ably.com/blog/scaling-pub-sub-with-websockets-and-redis)

**Used by:** Socket.IO Redis adapter, most moderate-scale real-time apps.

**Trade-off:** Simple, no extra infrastructure beyond Redis. Every message goes through Redis even if sender and receiver are on the same pod.

### Pattern 2: Connection Registry + Direct Routing (Netflix)

A global registry tracks which client is on which server. When a message arrives, look up the registry, find the server, route directly to that server.

> "Each Zuul Push server maintains a local, in-memory registry of all the clients connected to it. For multi-node clusters, this connects to a second level, off-the-box global datastore to enable cross-server lookups."
> — [Netflix Zuul Push Messaging Wiki](https://github.com/Netflix/zuul/wiki/Push-Messaging)

> "Zuul push allows you to plugin any datastore of your choice as the global push registry" with recommended features including low latency, TTL support, sharding, and replication. "Redis, Cassandra, Amazon DynamoDB are just few of many possible good choices."
> — same source

Netflix handles **5.5 million concurrent connections** at peak across multiple AWS regions with this pattern.

**Trade-off:** More efficient routing (direct server-to-server), but you build the registry lookup + direct routing yourself.

**References:**
- [Netflix Tech Blog: Open Sourcing Zuul 2](https://netflixtechblog.com/open-sourcing-zuul-2-82ea476cb2b3)
- [InfoQ: Scaling Push Messaging for Millions of Devices @Netflix](https://www.infoq.com/news/2018/07/zuul-push-messaging/)

### Pattern 3: Sharding / Client-Directed Routing (Discord)

Clients are told which specific server to connect to. No cross-server routing needed because the system pre-determines where each client lands.

Discord clients send an HTTP GET to a `/gateway` endpoint to receive the WebSocket URL to connect to. Guilds are sharded — each guild is assigned to a specific Elixir process, and messages within a guild never cross servers.

**Trade-off:** Most efficient (zero cross-server hops), but requires sharding logic and client cooperation.

**References:**
- [Discord Gateway Documentation](https://discord.com/developers/docs/events/gateway)

### Pattern 4: Sticky Sessions (Slack)

The load balancer pins each client to a specific server. All traffic for that client always goes to the same backend.

> "WebSocket connections start out as regular HTTPS connections, and then the client issues a protocol switch request to upgrade the connection to a websocket."
> — [Slack Engineering: Migrating Millions of Concurrent WebSockets to Envoy](https://slack.engineering/migrating-millions-of-concurrent-websockets-to-envoy/)

Slack deploys Envoy regionally with dynamically configured clusters so backends can be added or removed without reloading.

**Trade-off:** Simplest to implement, but fragile during deployments. WebSocket.org calls it ["a liability at scale due to cascading failures"](https://websocket.org/guides/websockets-at-scale/).

### Our choice: Pattern 1 (Pub/Sub Backplane)

| Pattern | Scale | Complexity | Fit for us |
|---|---|---|---|
| **Pub/Sub backplane** | Moderate (100s-1000s connections) | Low | **Yes — this one** |
| Connection registry + direct routing | Large (millions) | Medium | Overkill |
| Client-directed sharding | Very large (millions + multi-region) | High | Way overkill |
| Sticky sessions | Any (but fragile) | Lowest | Too fragile for rolling updates |

At our scale (tens to hundreds of mediator connections), the pub/sub backplane via Socket.IO's `RedisManager` / `AsyncRedisManager` is the standard answer. The adapter is built into `python-socketio` — no separate package. It uses Redis `PUBLISH`/`SUBSCRIBE` under the hood, with the `PubSubManager` base class providing the generic framework.

```python
# Flask-SocketIO — one line to enable cross-replica routing
socketio = SocketIO(app, message_queue='redis://redis:6379')
```

If we ever hit thousands of connections, Pattern 2 (Netflix-style registry) would be the next step.

**General references:**
- [WebSocket.org: WebSockets at Scale](https://websocket.org/guides/websockets-at-scale/)
- [Ably: The Challenge of Scaling WebSockets](https://ably.com/topic/the-challenge-of-scaling-websockets)
- [Ably: Scaling Pub/Sub with WebSockets and Redis](https://ably.com/blog/scaling-pub-sub-with-websockets-and-redis)
- [python-socketio Redis manager source](https://github.com/miguelgrinberg/python-socketio/blob/main/src/socketio/redis_manager.py)
- [python-socketio server docs](https://python-socketio.readthedocs.io/en/latest/server.html)

---

## Local Development

**Core principle**: Relay stays on the cluster (always deployed, shared infra). You test individual pieces locally by connecting to the deployed relay via Telepresence or `kubectl port-forward`. No need to recreate the whole pipeline locally.

### Reaching the relay from your laptop

```bash
# Option 1: Telepresence (you already use this)
telepresence connect
# K8s internal DNS works from your laptop:
# ws://archie-service-relay.default.svc.cluster.local:8081

# Option 2: kubectl port-forward
kubectl port-forward svc/archie-service-relay 8081:8081
# ws://localhost:8081 reaches the relay
```

### Mode A: Daily development (Telepresence — same as today)

No relay, no WebSocket. `WS_RELAY_URL` not set → Flask HTTP routes active.

```
Your laptop (Telepresence)           Customer K8s
┌──────────────┐                     ┌──────────────┐
│ Mediator     │←──intercept────────→│ Redis        │
│ (HTTP mode)  │                     │ Worker pods  │
│              │                     └──────────────┘
│ Chrome       │
│ (local)      │  ← chrome-devtools-mcp connects directly
└──────────────┘
```

- Mediator logic, runner/command ops: works as today
- Chrome MCP: uses local Chrome on laptop (existing fallback behavior)
- No relay, no tunnel, no docker-compose needed

### Mode B: Test individual pieces against cluster relay

Run only the component you're developing locally. Everything else stays on the cluster.

```
Your laptop                          Customer K8s cluster
┌─────────────────────┐              ┌──────────────────────────┐
│                     │              │                          │
│  Component under    │  WSS / HTTP  │  Relay (:8081)           │
│  test (local)       ├─────────────→│       │                  │
│                     │  via         │       ▼                  │
│                     │  Telepresence│  Mediator                │
│                     │  or port-fwd │       │                  │
│                     │              │       ▼                  │
│                     │              │  Worker (+ Chrome)       │
└─────────────────────┘              └──────────────────────────┘
```

**What you can test locally against the cluster relay:**

| What you're testing | What runs locally | What stays on cluster |
|---|---|---|
| TunnelProxy only | TunnelProxy (localhost:9222) | Relay, Mediator, Worker |
| Mediator WS client | Mediator (via Telepresence) | Relay, Worker |
| SDK through relay | Test script with BlitzyClient | Relay, Mediator, Worker |
| Full tunnel + SDK | TunnelProxy + BlitzyClient | Relay, Mediator, Worker |

**Example — testing TunnelProxy locally:**

```python
from blitzy_utils.runner_ops.tunnel import TunnelManager

# Connect to relay on cluster via port-forward or Telepresence
mgr = TunnelManager(relay_url="ws://localhost:8081", runner_id="runner-abc123")
mgr.start()
mgr.open(remote_port=9222, local_port=9222)

# chrome-devtools-mcp connects to localhost:9222
# Traffic tunnels: laptop → relay → mediator → worker Chrome
```

No port conflict because worker Chrome is remote. Your laptop owns `localhost:9222`.

### Mode C: Full integration testing (docker-compose)

When you need to test the entire pipeline end-to-end without any cluster dependency:

```yaml
services:
  redis:
    image: redis:7
    ports: ["6379:6379"]
  relay:
    build: ./archie-service-relay
    ports: ["8081:8081"]
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - REDIS_USERNAME=
      - REDIS_PASSWORD=
    depends_on: [redis]
  mediator:
    build: ./archie-client-mediator
    environment:
      - WS_RELAY_URL=ws://relay:8081
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    depends_on: [redis, relay]
  worker:
    build: ./archie-client-worker
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - RUNNER_CAPABILITIES=["CHROME"]
    depends_on: [redis]
```

Tests the full path: SDK → relay HTTP → relay WS → mediator → Redis → worker.

### When to use which

| Scenario | Mode | What's local | Relay via |
|---|---|---|---|
| Daily mediator debugging | A (Telepresence) | Mediator | N/A (HTTP mode) |
| Test tunnel / SDK / WS client | B (piece-by-piece) | Component under test | Telepresence / port-forward |
| Full end-to-end without cluster | C (docker-compose) | Everything | docker-compose |
| Staging / production | K8s deployment | Nothing | K8s internal |

---

## Libraries

| Component | Library | Purpose |
|---|---|---|
| Relay server | `Flask-SocketIO` + `gevent` | Flask + Socket.IO server (WS connections, namespaces, rooms) |
| Relay HTTP API | `Flask` | HTTP API for SDK requests + health endpoint (same Flask app) |
| Relay multi-replica | `Flask-SocketIO` Redis message queue | Cross-replica message routing via Redis pub/sub |
| Relay log/metric forwarding | `datadog-api-client` | Forward logs and metrics to Datadog Intake APIs |
| Mediator WS client | `python-socketio` | Socket.IO client (connect to relay) |
| Mediator CDP forwarder | `websockets` | Raw WS connections to worker Chrome instances |
| Consumer CDP proxy | `websockets` | Lightweight local proxy on localhost:9222 |

---

## Implementation Details — Relay Service

### Starting point

`archie-service-relay` is currently a blank Flask template deployed on Cloud Run (Knative). It has health check routes, Gunicorn, `blitzy-utils`, `blitzy-flask-utils`, and standard error handling. No Socket.IO, no Redis, no relay logic.

### Module structure

```
archie-service-relay/
├── main.py                                    # Flask + Flask-SocketIO init, DI wiring
├── requirements.txt                           # Add: flask-socketio, gevent, gevent-websocket, redis
├── Dockerfile                                 # Change: gevent worker class
├── swagger.yaml                               # API schema — models generated via `make pre-setup`
├── src/
│   ├── consts.py                              # Env vars: REDIS_HOST/PORT/USERNAME/PASSWORD, PORT, LOG_LEVEL
│   ├── api/
│   │   ├── models.py                          # AUTO-GENERATED from swagger.yaml via `make pre-setup`
│   │   └── routes/
│   │       ├── utils.py                       # Health checks (existing)
│   │       ├── relay.py                       # Transparent HTTP→WS proxy (X-Client-ID)
│   │       └── connections.py                 # GET /health/connections
│   ├── error/                                 # EXISTS — keep as-is
│   │   ├── base_error.py
│   │   └── errors.py
│   ├── services/
│   │   ├── interfaces/
│   │   │   ├── i_auth_service.py              # ABC — authenticate(api_key) -> client_id
│   │   │   └── i_registry_service.py          # ABC — register/unregister/is_online/get_all
│   │   ├── auth_service.py                    # AuthService(IAuthService)
│   │   └── registry_service.py                # RegistryService(IRegistryService) — Redis CRUD
│   ├── namespaces/
│   │   ├── control.py                         # /control — takes IAuthService + IRegistryService
│   │   ├── tunnel.py                          # /tunnel — generic port forwarding
│   │   ├── logs.py                            # /logs — log batch → Datadog (future)
│   │   └── metrics.py                         # /metrics — metric batch → Datadog (future)
│   ├── utils/
│   │   ├── interfaces/
│   │   │   └── i_redis_client_provider.py     # ABC — get_redis_client()
│   │   └── redis_client_provider.py           # RedisClientProvider(IRedisClientProvider)
│   └── models_config/                         # EXISTS — keep as-is
```

**Conventions:**
- All API models (`src/api/models.py`) are auto-generated from `swagger.yaml` via `make pre-setup`. Do NOT write models manually.
- All services and utils follow interface-based design (ABC) for dependency injection. No tests are written as part of this plan, but all code must be structured so fakes/mocks can be injected at any time.
- Redis env vars (`REDIS_HOST`, `REDIS_PORT`, `REDIS_USERNAME`, `REDIS_PASSWORD`) exposed via `src/consts.py` from environment.
- Designed for Kubernetes deployment.

### New dependencies

```
# requirements.txt additions
flask-socketio>=5.3.0
gevent>=24.0.0
gevent-websocket>=0.10.1
redis>=5.0.0
```

### Key files to build

**`main.py`** — Flask + Flask-SocketIO with gevent + Redis adapter:

```python
from flask import Flask
from flask_socketio import SocketIO

app = Flask(__name__)
socketio = SocketIO(
    app,
    message_queue=redis_url,                        # Redis adapter for cross-replica (constructed from REDIS_HOST/PORT/USERNAME/PASSWORD)
    async_mode='gevent',                            # gevent for production
    max_http_buffer_size=10 * 1024 * 1024,          # 10 MB (CDP screenshots)
    cors_allowed_origins='*',
)

# Register HTTP routes (blueprints)
app.register_blueprint(utils_bp)
app.register_blueprint(relay_bp)
app.register_blueprint(connections_bp)

# Register Socket.IO namespaces
socketio.on_namespace(ControlNamespace('/control'))
socketio.on_namespace(TunnelNamespace('/tunnel'))
```

**`src/auth.py`** — Resolves API key to `client_id` (`ClientInstallation.id`):

```python
class AuthService:
    """Validate API key and return the client_id it belongs to."""

    def authenticate(self, api_key: str) -> str:
        """Returns ClientInstallation.id for this API key.

        Implementation options (choose one):
        A. Call admin service: GET /v1/client/auth/validate → client_id
        B. JWT-encoded API key: decode → extract client_id claim
        C. Redis cache: api_key_hash → client_id (populated by admin service)
        """
```

**`src/registry.py`** — Connection state in Redis:

```python
class ConnectionRegistry:
    """Tracks which mediators are connected and on which socket."""

    async def register(self, client_id: str, sid: str): ...
    async def unregister(self, client_id: str): ...
    async def is_online(self, client_id: str) -> bool: ...
    async def get_all(self) -> dict: ...  # For /health/connections
```

**`src/api/routes/relay.py`** — Transparent HTTP→WS proxy:

The relay accepts **any HTTP path** when `X-Client-ID` header is present. This means callers (SDK, admin service, any future service) just send normal HTTP requests to the relay URL — same paths they'd send to the mediator directly — and the relay wraps and forwards via WS. No special relay protocol for callers to learn.

```python
@relay_bp.route('/', defaults={'path': ''}, methods=['GET', 'POST', 'PUT', 'DELETE'])
@relay_bp.route('/<path:path>', methods=['GET', 'POST', 'PUT', 'DELETE'])
def relay_request(path):
    client_id = request.headers.get('X-Client-ID')
    if not client_id:
        return jsonify({"error": "Missing X-Client-ID"}), 400
    if not registry.is_online(client_id):
        return jsonify({"error": "Mediator offline"}), 503

    payload = {
        "method": request.method,
        "path": f"/{path}",
        "body": request.get_json(silent=True),
        "query": dict(request.args),
    }
    response = socketio.call(
        'request', payload,
        room=client_id, namespace='/control', timeout=30,
    )
    return jsonify(response)
```

**Why transparent**: Admin service pushes secrets to `relay_url/api/v1/secrets/all` — same path it uses today against the mediator, just different host + `X-Client-ID` header. SDK sends to `relay_url/api/v1/runners` — same pattern. No one needs to wrap payloads in a relay-specific format.

**`src/namespaces/control.py`** — Mediator registration + request routing:

```python
class ControlNamespace(Namespace):
    def on_register(self, data):
        client_id = auth_service.authenticate(data['api_key'])
        join_room(client_id)
        registry.register(client_id, request.sid)

    def on_disconnect(self):
        client_id = self._get_client_id(request.sid)
        registry.unregister(client_id)
```

**`src/namespaces/tunnel.py`** — Generic port forwarding (stateless relay):

```python
class TunnelNamespace(Namespace):
    def on_tunnel_data(self, data):
        """Forward frame to the other side (consumer ↔ mediator)."""
        runner_id = data['runner_id']
        emit('tunnel_data', data, room=runner_id, include_self=False)
```

### Dockerfile change

```dockerfile
# BEFORE (Cloud Run, threaded)
CMD exec gunicorn --bind :$PORT --workers 1 --threads 8 --timeout 0 main:app

# AFTER (K8s, gevent WebSocket)
CMD exec gunicorn --bind :$PORT \
    --worker-class geventwebsocket.gunicorn.workers.GeventWebSocketWorker \
    --workers 1 \
    --timeout 0 \
    main:app
```

Single worker because gevent handles concurrency via greenlets. Flask-SocketIO requires this — multiple Gunicorn workers would break WebSocket state.

### Deployment change — Knative → K8s Deployment

The relay currently uses `service.yaml` (Knative/Cloud Run). This must be replaced with a standard K8s Deployment + Service because Knative has a 60-minute WebSocket timeout and scales to zero (killing persistent connections).

```yaml
# BEFORE: service.yaml (Knative)
apiVersion: serving.knative.dev/v1
kind: Service
# ... Knative spec with timeoutSeconds: 300

# AFTER: K8s Deployment + Service (new helm chart or manifest)
# See Helm Chart Changes section below
```

### Estimated size

| Module | Lines | Complexity |
|---|---|---|
| `main.py` (rewrite) | ~50 | Flask-SocketIO init, blueprint/namespace registration |
| `auth.py` | ~40 | API key validation |
| `registry.py` | ~60 | Redis CRUD for connection state |
| `relay.py` (HTTP route) | ~30 | HTTP→WS bridge |
| `connections.py` (health) | ~20 | List connected mediators |
| `control.py` (namespace) | ~80 | Register, disconnect, request routing |
| `tunnel.py` (namespace) | ~60 | Frame forwarding |
| `logs.py` + `metrics.py` | ~100 | Datadog forwarding (future) |
| **Total** | **~440** | |

---

## Implementation Details — Mediator Changes

### Current mediator structure

```
Flask Application (main.py)
  ├── Blueprints (REST HTTP Routes)
  │   ├── /commands:     submit_command, get_status, restart_session
  │   ├── /runners:      create_runner, delete_runner, get_runner_status
  │   ├── /secrets:      CRUD operations
  │   ├── /registration: register mediator with main server
  │   └── /utils:        health, uptime, redis checks
  │
  ├── Service Layer (unchanged with WS migration)
  │   ├── CommandService      (submit, poll, restart)
  │   ├── RunnerService       (create, delete, status)
  │   ├── RedisQueueService   (queue lifecycle)
  │   ├── KubernetesService   (K8s API operations)
  │   ├── BlitzyService       (main server comms)
  │   └── SecretsService      (vault operations)
  │
  ├── Repository Layer (unchanged)
  │   ├── CommandExecutionRepository
  │   ├── JobServerRepository
  │   ├── CallbackQueueRepository
  │   └── CommandArchiveRepository
  │
  └── RQ Worker subprocess (unchanged)
```

### New files

**`src/services/ws_client.py`** — Socket.IO client that connects outbound to the relay:

```python
class WSClient:
    """Outbound Socket.IO connection to the relay service."""

    def __init__(self, relay_url: str, api_key: str):
        self._sio = socketio.Client(
            reconnection=True,
            reconnection_attempts=0,       # infinite
            reconnection_delay=1,
            reconnection_delay_max=5,
        )
        self._relay_url = relay_url
        self._api_key = api_key
        self._router = WSRouter()
        self._register_handlers()

    def _register_handlers(self):
        @self._sio.on('request', namespace='/control')
        def handle_request(data):
            return self._router.dispatch(data)

        @self._sio.event(namespace='/control')
        def connect():
            self._sio.emit('register', {'api_key': self._api_key}, namespace='/control')

    def start(self):
        self._sio.connect(
            self._relay_url,
            namespaces=['/control', '/tunnel'],
        )

    def stop(self):
        self._sio.disconnect()
```

**`src/services/ws_router.py`** — Dispatches incoming WS messages to existing service methods:

```python
class WSRouter:
    """Routes WS messages to existing service layer.

    The service layer (CommandService, RunnerService, etc.) stays unchanged.
    This router replaces Flask's URL routing for WS mode.
    """

    def dispatch(self, message: dict) -> dict:
        """Route a relayed request to the correct service method.

        Message format from relay:
        {
            "method": "POST",
            "path": "/api/v1/runners",
            "body": { ... },
            "headers": { ... }
        }
        """
        method = message['method']
        path = message['path']
        body = message.get('body', {})

        # Runner routes
        if method == 'POST' and path == '/api/v1/runners':
            return self._create_runner(body)
        if method == 'GET' and path.startswith('/api/v1/runners/'):
            runner_id = path.split('/')[-1]
            return self._get_runner_status(runner_id)
        if method == 'DELETE' and path.startswith('/api/v1/runners/'):
            runner_id = path.split('/')[-1]
            return self._delete_runner(runner_id)

        # Command routes
        if method == 'POST' and '/commands' in path:
            job_id = path.split('/')[4]  # /api/v1/jobs/{id}/commands
            return self._submit_command(job_id, body)
        if method == 'GET' and '/status' in path:
            exec_id = path.split('/')[4]  # /api/v1/commands/{id}/status
            return self._get_command_status(exec_id)

        # ... secrets, restart-session, etc.
```

**`src/services/tunnel_forwarder.py`** — Forwards tunnel frames from relay to worker services:

```python
class TunnelForwarder:
    """Forwards tunnel frames between relay and worker K8s services.

    Manages raw TCP connections to worker pods (e.g., Chrome on :9222).
    One connection per (runner_id, port, connection_id) tuple.
    """

    def __init__(self, sio_client):
        self._sio = sio_client
        self._connections = {}  # (runner_id, port, conn_id) → (reader, writer)

    async def handle_open(self, runner_id, port, connection_id):
        """Open TCP connection to worker service."""
        host = f"chrome-svc-{runner_id}.{K8S_NAMESPACE}.svc.cluster.local"
        reader, writer = await asyncio.open_connection(host, port)
        self._connections[(runner_id, port, connection_id)] = (reader, writer)
        asyncio.create_task(self._read_loop(runner_id, port, connection_id, reader))

    async def handle_data(self, runner_id, port, connection_id, data):
        """Forward data from relay to worker."""
        _, writer = self._connections[(runner_id, port, connection_id)]
        writer.write(data)
        await writer.drain()

    async def handle_close(self, runner_id, port, connection_id):
        """Close TCP connection to worker."""
        ...
```

### Modified files

**`main.py`** — Conditional WS client startup (no new env var — uses existing `MAIN_SERVER_URL`):

```python
# Add after existing Flask setup:
from src.consts import MAIN_SERVER_URL, API_KEY

if MAIN_SERVER_URL:
    relay_url = f"{MAIN_SERVER_URL}/v1/relay"
    from src.services.ws_client import WSClient
    ws_client = WSClient(relay_url=relay_url, api_key=API_KEY)
    ws_client.start()
    logger.info("ws_client_started", relay_url=relay_url)
else:
    logger.info("ws_mode_disabled", reason="MAIN_SERVER_URL not set")
```

Flask routes, RQ worker, health checks — all stay. The WS client runs alongside them.

**`src/consts.py`** — Remove gateway consts (no new env vars needed):

```python
# REMOVE (after Phase 4)
# GATEWAY_HOST = os.environ.get("GATEWAY_HOST")
# GATEWAY_NAME = os.environ.get("GATEWAY_NAME")
# BLITZY_CLIENT_URL = os.environ.get("BLITZY_CLIENT_URL")

# MAIN_SERVER_URL already exists — relay is at MAIN_SERVER_URL/v1/relay
# API_KEY already exists — used for WS auth handshake
```

**`src/services/blitzy_service.py`** — Transport change for outbound calls:

```python
# BEFORE: HTTP calls to main server
class BlitzyService(IBlitzyService):
    def register(self):
        response = self._http_client.put(f"{MAIN_SERVER_URL}/v1/client/register", json=payload)
        ...

# AFTER: WS messages via Socket.IO client (when WS_RELAY_URL set)
class BlitzyService(IBlitzyService):
    def register(self):
        if self._ws_client:
            # Registration is implicit on WS connect — skip explicit HTTP call
            return
        # Fallback: HTTP (local dev mode)
        response = self._http_client.put(f"{MAIN_SERVER_URL}/v1/client/register", json=payload)
```

**`src/services/kubernetes_service.py`** — Remove CDP route methods (Phase 4):

```python
# REMOVE (Phase 4 — after tunnel is verified working)
# def create_chrome_route(self, runner_id, namespace): ...
# def delete_chrome_route(self, runner_id, namespace): ...
# def _build_chrome_service_name(self, runner_id): ...
# def _build_chrome_route_name(self, runner_id): ...

# KEEP — Chrome K8s ClusterIP Service creation (needed for tunnel)
# def create_service(...): ...  # Still creates chrome-svc-{runner_id}
```

**`src/services/runner_service.py`** — Remove CDP route calls (Phase 4):

```python
# In create_runner():
# REMOVE: self._kubernetes_service.create_chrome_route(runner_id, namespace)
# KEEP:   self._kubernetes_service.create_service(...)  # ClusterIP for tunnel

# In delete_runner():
# REMOVE: self._kubernetes_service.delete_chrome_route(runner_id, namespace)

# capabilities_metadata["chrome_url"]:
# BEFORE: "https://chaitanya-client.blitzy.dev/cdp/runner-abc123"
# AFTER:  runner_id (the tunnel uses runner_id to route, not a URL)
```

### What stays completely unchanged

| Component | Reason |
|---|---|
| `CommandService` | WS Router calls its methods exactly as Flask routes do |
| `RunnerService` (core logic) | Only CDP route calls removed; create/delete/status unchanged |
| `RedisQueueService` | No relation to WS — mediator↔worker still uses Redis queues |
| `SecretsService` | Vault operations unchanged |
| All repositories | Data access layer untouched |
| All Pydantic models | Request/response schemas unchanged |
| RQ worker subprocess | Async job processing unchanged |
| Health check routes | Flask stays for K8s probes |

---

## Implementation Details — Helm Chart Changes

### Current helm chart structure (`archie-helm-chart/blitzy-client/chart/`)

The mediator is deployed via a Helm chart that includes:
- `client-mediator.yaml` — Deployment + ClusterIP Service (port 8080)
- `gateway.yaml` — Gateway + HTTPRoute (Envoy, HTTPS on `chaitanya-client.blitzy.dev`)
- `gatewayclass.yaml` — GatewayClass for Envoy
- `envoyproxy.yaml` — EnvoyProxy configuration
- `certificate.yaml` — Let's Encrypt TLS cert
- `cluster-issuer.yaml` — ACME cluster issuer
- `configmap.yaml` — Environment variables
- `service-account.yaml` — RBAC
- Dependencies: cert-manager, gateway-helm (Envoy), redis, cloudnative-pg, vault, otel

### Changes to existing chart

**No changes to blitzy-client chart.** Envoy Gateway stays as-is — it serves its own purpose. All existing config (`BLITZY_CLIENT_URL`, `GATEWAY_HOST`, `GATEWAY_NAME`, gateway-helm dependency, all templates) remains unchanged.

No new env vars needed — `MAIN_SERVER_URL` and `API_KEY` already exist.

### New chart: Relay Service

The relay needs its own Helm chart or K8s manifests (replacing the Knative `service.yaml`):

```yaml
# relay-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: archie-service-relay
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: relay
          image: {{ .Values.relay.image }}
          ports:
            - containerPort: 8080
          env:
            - name: REDIS_HOST
              value: {{ .Values.relay.redis.host }}
            - name: REDIS_PORT
              value: {{ .Values.relay.redis.port | quote }}
            - name: REDIS_USERNAME
              value: {{ .Values.relay.redis.username }}
            - name: REDIS_PASSWORD
              value: {{ .Values.relay.redis.password }}
            - name: PORT
              value: "8080"
            - name: LOG_LEVEL
              value: {{ .Values.relay.logLevel }}
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 1000m
              memory: 1Gi
          livenessProbe:
            httpGet:
              path: /v1/health-check
              port: 8080
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /v1/health-check
              port: 8080
            periodSeconds: 10
          lifecycle:
            preStop:
              exec:
                command: ["python", "-c", "import time; time.sleep(30)"]
---
# relay-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: archie-service-relay
spec:
  type: ClusterIP
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: archie-service-relay
```

The relay is deployed on **Blitzy K8s** (not customer K8s). It can use the existing `helm-chart/` generic template as a base, or be a standalone chart in the `archie-helm-chart` repo.

### RBAC changes

No RBAC changes needed. Envoy Gateway stays, so the mediator's service account keeps its existing permissions (including Gateway API CRDs).

### Phased rollout in Helm

| Phase | Helm change |
|---|---|
| Phase 1 | Deploy relay service on Blitzy K8s. Add `/v1/relay` HTTPRoute rule to `api-gateway` chart. Add GCPBackendPolicy for WebSocket timeout. No changes to blitzy-client chart (mediator already has `MAIN_SERVER_URL`). |
| Phase 2 | No Helm changes (SDK + admin service code changes only). |
| Phase 3 | No Helm changes (mediator code + consumer code). |
| Phase 4 | No Helm changes. Envoy Gateway stays as-is. CDP route creation code removed from mediator (code change, not Helm). |

---

## Implementation Details — Gateway Routing (Path-Prefix Pattern)

### Pattern: Same as chat service

The chat service (`archie-service-chat`) is already routed via GKE Gateway API with path-prefix matching:

```yaml
# Existing chat service routing (api-gateway/values.yaml)
rules:
  - matches:
    - path:
        type: PathPrefix
        value: /v1/api/chat          # Match this prefix
    backendRefs:
    - kind: Service
      name: archieservicechat-dev
      port: 8080
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          replacePrefixMatch: /       # Strip prefix before forwarding
          type: ReplacePrefixMatch
```

The relay service uses the **exact same pattern** — a new HTTPRoute rule on the existing `os.api-k.blitzy.dev` gateway:

```yaml
# New relay service routing rule
- matches:
  - path:
      type: PathPrefix
      value: /v1/relay               # All relay traffic
  backendRefs:
  - kind: Service
    name: archie-service-relay
    port: 8080
  filters:
  - type: URLRewrite
    urlRewrite:
      path:
        replacePrefixMatch: /         # Strip /v1/relay prefix
        type: ReplacePrefixMatch
```

### Request flow

```
Mediator (customer K8s)                    Blitzy Cloud (GKE)
┌──────────────────────┐                   ┌─────────────────────────────────────┐
│                      │                   │                                     │
│  MAIN_SERVER_URL     │    WebSocket      │  GKE Gateway (os.api-k.blitzy.dev) │
│  = os.api-k.blitzy   │──────────────────>│                                     │
│       .dev           │                   │  /v1/relay/*  → relay-service:8080  │
│                      │                   │  /v1/client/* → admin-service:8080  │
│  No relay URL needed │                   │  /v1/chat/*   → chat-service:8080  │
│                      │                   │                                     │
└──────────────────────┘                   └─────────────────────────────────────┘

SDK (Cloud Run consumer)                        │
┌──────────────────────┐                        │
│  BlitzyClient        │   HTTP + X-Client-ID   │
│  → MAIN_SERVER_URL   │───────────────────────>│
│    /v1/relay/...     │                        │
└──────────────────────┘                        │

Admin service (internal)                        │
┌──────────────────────┐                        │
│  BlitzyClient        │   Internal K8s svc     │
│  → relay-svc:8080    │───────────────────────>│  (or via gateway)
│    /api/v1/secrets   │                        │
└──────────────────────┘
```

### What each caller sees

| Caller | URL it connects to | How it knows |
|--------|-------------------|-------------|
| Mediator (WS connect) | `MAIN_SERVER_URL/v1/relay/socket.io` | `MAIN_SERVER_URL` already in Helm config. No new env var. |
| SDK (HTTP requests) | `MAIN_SERVER_URL/v1/relay/api/v1/runners` | Same `MAIN_SERVER_URL` from admin API, with `/v1/relay` prefix. |
| Admin service (secret push) | `relay-svc.namespace:8080/api/v1/secrets/all` | Internal K8s service discovery. No gateway needed. |

### Backend policy for WebSocket

The chat service uses `timeoutSec: 3600` (1 hour) for SSE. The relay needs a longer timeout for persistent WebSocket:

```yaml
apiVersion: networking.gke.io/v1
kind: GCPBackendPolicy
spec:
  default:
    timeoutSec: 86400            # 24 hours — WebSocket persistent connection
    connectionDraining:
      drainingTimeoutSec: 30     # Wait 30s during rolling updates
  targetRef:
    kind: Service
    name: archie-service-relay
```

GKE Gateway API supports WebSocket natively — the `Upgrade: websocket` header is handled automatically by `gke-l7-global-external-managed`.

### Key insight: No new URL for mediator

The mediator **already knows** `MAIN_SERVER_URL` (`os.api-k.blitzy.dev`). The relay is just another path under that same gateway. The mediator connects to `MAIN_SERVER_URL/v1/relay/socket.io` — no `WS_RELAY_URL` env var needed.

`BLITZY_CLIENT_URL` stays — Envoy Gateway serves its own purpose. The relay is an additional communication channel, not a replacement for the gateway.

---

## Implementation Details — Admin Service Changes (`archie-service-admin`)

### Impact summary

The admin service needs **minimal code changes**. Most flows are **mediator → admin** (outbound from mediator) which stay as direct HTTP. The only flow that breaks is **admin → mediator** (secret push) because the mediator no longer has a public URL.

The fix: admin service uses `BlitzyClient` from `blitzy-utils-python` to push secrets through the relay, replacing the internal `HttpForwardingClient`.

### What does NOT change

| Component | Why no change |
|-----------|--------------|
| `GET /v1/client/entity/{type}/{id}` | Already returns `client_id` at top level. Response shape unchanged. |
| `PUT /v1/client/register` | Mediator still calls this for heartbeat/re-registration. May stop sending URL field (no public URL to advertise). |
| `POST /v1/client/register` (initial setup) | Unchanged. Creates installation during onboarding. |
| `POST /v1/client/job/register` | Mediator → admin (outbound). Stays as direct HTTP with API key. |
| `GET /v1/client/job/{id}` | Mediator → admin (outbound). Stays as HTTP. |
| API key auth (`validate_api_key`) | Unchanged. Mediator still uses API key for direct HTTP calls to admin. |
| Client models, repository layer | Unchanged. |
| Default installation logic | `get_default_client_for_entity()` with 2-level fallback works as-is. |

### What DOES change: Secret forwarding uses `BlitzyClient`

**Today**: Admin pushes secrets using its own internal `HttpForwardingClient`:
```
Admin → HttpForwardingClient → POST mediator_url/api/v1/secrets/all → Mediator (direct)
```

**After**: Admin uses `BlitzyClient` (from `blitzy-utils-python`) to push through relay:
```
Admin → BlitzyClient → POST relay_svc/api/v1/secrets/all (X-Client-ID) → Relay → WS → Mediator
```

Since admin is internal to the Blitzy K8s cluster, it reaches the relay via internal K8s service URL (no gateway needed).

#### Changes in `src/service/env_exchange_service.py`

```diff
+ from blitzy_utils.blitzy_client import BlitzyClient

  class EnvExchangeService:
-     def __init__(self, http_client=None):
-         self.http_client = http_client or HttpForwardingClient()
+     def __init__(self, blitzy_client=None):
+         self._blitzy_client = blitzy_client or BlitzyClient()

      def _forward_secrets_to_client(self, context, gsm_secrets):
-         forwarding_result = self.http_client.forward_secrets(
-             context.forwarded_to_url, massaged_payload
-         )
+         # Use BlitzyClient to push through relay
+         # BlitzyClient handles X-Client-ID header injection
+         client_id = str(context.client.id)
+         self._blitzy_client.push_to_mediator(
+             client_id=client_id,
+             path="/api/v1/secrets/all",
+             payload=massaged_payload,
+         )
```

`BlitzyClient` may expose a `push_to_mediator(client_id, path, payload)` method for this use case — a direct relay call without needing `get_client_url()` resolution. The exact API can be refined during implementation.

#### What gets removed from admin service

| File | Change |
|------|--------|
| `src/clients/http_forwarding_client.py` | **Removed entirely** — replaced by `BlitzyClient` |
| `src/service/env_exchange_service.py` | Replace `HttpForwardingClient` with `BlitzyClient` |
| `requirements.txt` | Add `blitzy-utils` dependency (for `BlitzyClient`) |

### Unified SDK — all relay-bound traffic through `BlitzyClient`

```
                              All mediator-bound traffic
                              uses BlitzyClient SDK
                              (from blitzy-utils-python)
                                      │
         ┌──────────────────────────────┼──────────────────────────────┐
         │                              │                              │
    Consumer Job                  Admin Service                   Future Service
    (create_runner,               (push_secrets)                  (anything)
     submit_command)                    │                              │
         │                              │                              │
         ▼                              ▼                              ▼
    BlitzyClient                  BlitzyClient                  BlitzyClient
    X-Client-ID: abc              X-Client-ID: abc              X-Client-ID: abc
         │                              │                              │
         │ via gateway                  │ via internal K8s svc         │
         └──────────────────────────────┼──────────────────────────────┘
                                        │
                                        ▼
                                   Relay Service
                              (transparent HTTP proxy)
                                        │
                                        ▼ WS
                                     Mediator
```

### Admin service changes per migration phase

| Phase | Admin change |
|-------|-------------|
| Phase 1 | None. Admin still pushes directly to mediator URL. |
| Phase 2 | Replace `HttpForwardingClient` with `BlitzyClient`. Add `blitzy-utils` to requirements. Secret push routes through relay. |
| Phase 3 | None. CDP tunnel is mediator ↔ relay ↔ consumer. |
| Phase 4 | Remove `HttpForwardingClient`. Envoy Gateway and `BLITZY_CLIENT_URL` stay. |
| Phase 5 | Optional: relay pushes connection status events to admin (replace polling heartbeat). |

---

## Files Summary

### New files

| File | Repo | Description |
|---|---|---|
| Entire relay service | NEW repo | Socket.IO server + HTTP API + Redis registry |
| `src/services/ws_client.py` | archie-client-mediator | Socket.IO client to relay |
| `src/services/ws_router.py` | archie-client-mediator | Message dispatcher to existing services |
| `src/services/cdp_forwarder.py` | archie-client-mediator | CDP frame forwarding to worker Chrome |
| `runner_ops/cdp_proxy.py` | blitzy-utils-python | Local CDP proxy for consumer |
| `docker-compose.yml` | archie-client-mediator | Integration testing setup |

### Modified files

| File | Repo | Change |
|---|---|---|
| `blitzy_client.py` | blitzy-utils-python | Add `X-Client-ID` header, unified relay SDK (used by consumers + admin service) |
| `src/services/blitzy_service.py` | archie-client-mediator | Registration sends no URL (mediator has no public URL). Outbound calls stay HTTP. |
| `main.py` | archie-client-mediator | WS client connects to `MAIN_SERVER_URL/v1/relay/socket.io` |
| `src/services/kubernetes_service.py` | archie-client-mediator | Remove `create_chrome_route`, `delete_chrome_route` |
| `src/services/runner_service.py` | archie-client-mediator | Remove CDP route creation/cleanup |
| `src/consts.py` | archie-client-mediator | No removals — all gateway vars stay. No new env vars needed. |
| `src/service/env_exchange_service.py` | archie-service-admin | Replace `HttpForwardingClient` with `BlitzyClient` for secret push |
| `api-gateway/values.yaml` | archie-helm-chart | Add `/v1/relay` path-prefix rule + backend policy for relay service |
| `lib/blitzy/helper.py` | archie-job-reverse-code-generator | Use local CDP proxy instead of `--browserUrl` |

### Removed / disabled

| File | Repo | Change |
|---|---|---|
| `src/clients/http_forwarding_client.py` | archie-service-admin | **Removed** — replaced by `BlitzyClient` |
| `i_kubernetes_service.py` | archie-client-mediator | Remove abstract CDP route methods (`create_chrome_route`, `delete_chrome_route`) |
| `kubernetes_service.py` | archie-client-mediator | Remove CDP route methods (gateway itself stays) |

### Unchanged (Envoy Gateway stays)

| File | Repo | Why |
|---|---|---|
| `chart/values.yaml` | archie-helm-chart (blitzy-client) | Gateway stays enabled, all config stays |
| `chart/templates/gateway.yaml` | archie-helm-chart (blitzy-client) | Gateway serves its own purpose |
| `chart/templates/certificate.yaml` | archie-helm-chart (blitzy-client) | TLS cert still needed |
| `chart/templates/envoyproxy.yaml` | archie-helm-chart (blitzy-client) | Envoy config still needed |
| `i_kube_client_provider.py` | archie-client-mediator | `get_custom_objects_api()` stays |
| `kube_client_provider.py` | archie-client-mediator | `get_custom_objects_api()` stays |

---

## Migration Strategy

### Phase 1: Build relay + gateway routing (dual-mode)
- Build the relay service, deploy on Blitzy K8s
- Add `/v1/relay` path-prefix HTTPRoute rule to `os.api-k.blitzy.dev` GKE Gateway (same pattern as `/v1/api/chat`)
- Add GCPBackendPolicy with long timeout for WebSocket persistent connections
- Add Socket.IO client to mediator — connects to `MAIN_SERVER_URL/v1/relay/socket.io` (no new env var)
- Flask HTTP routes remain active — both channels work
- No SDK changes yet, no CDP tunnel yet

### Phase 2: SDK + admin service routing through relay
- Update `BlitzyClient` in `blitzy-utils-python`: add `X-Client-ID` header, cache `client_id` from admin response
- SDK sends requests to `MAIN_SERVER_URL/v1/relay/...` with `X-Client-ID`
- Admin service replaces `HttpForwardingClient` with `BlitzyClient` for secret push (via internal K8s svc)
- Verify all runner/command operations work through relay → WS → mediator
- Verify default (BLITZY_SHARED) installation routing works for entities without a dedicated client
- Verify secret push works: admin → BlitzyClient → relay → WS → mediator
- Keep Flask routes as fallback

### Phase 3: CDP tunnel
- Build CDP forwarder in mediator
- Build local CDP proxy in consumer
- Test Chrome MCP through tunnel end-to-end
- Verify screenshots, navigation, page evaluation work

### Phase 4: Remove CDP route code + cleanup
- Remove dynamic K8s CDP routing code from mediator (`create_chrome_route`, `delete_chrome_route`)
- Remove `HttpForwardingClient` from admin service (replaced by BlitzyClient in Phase 2)
- Envoy Gateway stays as-is — serves its own purpose
- Flask routes stay active alongside WS client

### Phase 5: Push optimization (future, optional)
- Mediator pushes status change events through WS (runner ready, command done)
- Replace SDK polling with push notification subscription
- Relay pushes connection status events to admin (replace polling heartbeat)
- Reduces load and improves response time

---

## Memory, Packet Sizing & Scaling

### Per-Connection Memory

| Component | Per connection | Notes |
|-----------|---------------|-------|
| Raw WebSocket (with compression) | ~64 KiB | Default `websockets` library baseline (`permessage-deflate` on) |
| Raw WebSocket (no compression) | ~14 KiB | Compression disabled |
| Socket.IO overhead | ~5-10 KiB | Session state, namespace membership, room mappings |
| TLS read buffer | ~256 KiB | For WSS connections |
| **Total per mediator (idle)** | **~330 KiB** | Steady-state with TLS + compression |

### Scaling Projections (Idle Connections)

Each customer cluster maintains **one** persistent WebSocket to the relay — not thousands of browser clients.

| Scale | Connections | Idle memory | Notes |
|-------|-------------|-------------|-------|
| 10 customers | 10 | ~3 MB | Negligible |
| 100 customers | 100 | ~33 MB | Still tiny |
| 1,000 customers | 1,000 | ~330 MB | Comfortable on a single relay replica |
| 10,000 customers | 10,000 | ~3.3 GB | Multi-replica territory |

Memory scales linearly with connection count. The bottleneck will be concurrent request throughput, not idle connections.

### Message / Packet Size Limits

| Setting | Default | Configured |
|---------|---------|-----------|
| `max_http_buffer_size` (python-socketio) | 1 MB | **10 MB** |
| WebSocket frame size (`websockets` `max_size`) | 1 MB | **10 MB** |
| Message queue depth | 16 frames (`max_queue`) | 16 frames (default) — up to 160 MB buffered per connection |

### What Flows Through the Socket

| Message type | Typical size | Peak size | Notes |
|-------------|-------------|-----------|-------|
| Runner CRUD (create/delete/status) | 1-5 KB | ~10 KB | Always within defaults |
| Command submit | 1-10 KB | ~50 KB | Large shell commands |
| Command status response (stdout/stderr) | 5-100 KB | 1-5 MB | Large build output |
| CDP screenshot (base64 PNG) | 500 KB - 2 MB | **5 MB** | Full-page high-res — the pinch point |
| CDP navigation/evaluation | 1-10 KB | ~50 KB | Always within defaults |

CDP screenshots are the largest payloads. A full-page screenshot at high resolution can reach 3-5 MB base64. The 10 MB configured limit provides comfortable headroom.

### Configuration

```python
# Relay server
sio = socketio.AsyncServer(
    async_mode='aiohttp',
    max_http_buffer_size=10 * 1024 * 1024,  # 10 MB
)

# Mediator client
sio = socketio.Client(
    websocket_extra_options={"max_size": 10 * 1024 * 1024},  # 10 MB
)
```

### Active Memory During Requests

When a request is in-flight, the memory spike per connection is:

```
Idle connection (~330 KB)
  + Request payload buffered (~5 KB for control, ~5 MB for CDP screenshot)
  + Response payload buffered (same range)
  ≈ Up to ~10 MB peak per connection during a CDP screenshot round-trip
```

This is transient — freed as soon as the ack completes. Worst case: 100 concurrent mediators each doing a CDP screenshot simultaneously → ~1 GB peak. Very manageable.

### OS-Level Limits

| Limit | Default | Recommended |
|-------|---------|-------------|
| File descriptors (`ulimit -n`) | 1,024 | 65,535 (each WS = 1 fd) |
| Local port range | ~28,000 | 10,000-65,535 (~55,000 connections per IP) |

### Scaling Recommendations

| Concern | Mitigation |
|---------|-----------|
| Message too large | `max_http_buffer_size=10MB`, use msgpack parser for binary efficiency |
| Large stdout | Truncate stdout/stderr at 1 MB on mediator side before sending (already done in current code) |
| CDP screenshot pressure | Infrequent — LLM decides when to screenshot, ~1-2 per minute per job |
| Connection limits (OS) | Raise ulimit to 65,535 on relay pods. Each mediator = 1 connection |
| Multi-replica relay | Socket.IO Redis adapter (`AsyncRedisManager`) handles cross-replica routing. Scale horizontally by adding replicas |
| Backpressure | If mediator is slow, Socket.IO queues up to 16 frames (160 MB at 10 MB limit). Beyond that, connection drops — SDK retries on 503 |

---

## Key Design Decisions

1. **Relay on K8s, not Cloud Run** — avoids Cloud Run's 60-minute WebSocket timeout. Persistent connection, no reconnect cycling.
2. **Socket.IO over raw WebSocket** — gives auto-reconnect, heartbeat, namespaces, rooms, binary support, ack callbacks for free. ~300 lines of infrastructure code avoided.
3. **Flask-SocketIO for relay** — keeps consistency with existing Flask tooling and `blitzy-flask-utils` library. Flask-SocketIO has no inherent connection limit ([confirmed by author](https://github.com/miguelgrinberg/Flask-SocketIO/issues/859)). Gevent backend recommended for production ([author's guidance](https://github.com/miguelgrinberg/Flask-SocketIO/discussions/2037)).
4. **HTTP and WebSocket on same server** — at our scale (tens to hundreds of connections), a single process handles both comfortably. A single server can support 240K+ concurrent connections ([Ably benchmark](https://ably.com/topic/the-challenge-of-scaling-websockets)). Separate only when load characteristics become incompatible ([WebSocket.org](https://websocket.org/guides/websockets-at-scale/)).
5. **Four namespaces** — `/control` (SDK ops), `/cdp` (Chrome tunnel), `/logs` (log forwarding), `/metrics` (metrics forwarding). All multiplexed on one WebSocket per mediator. No limit on namespaces or rooms ([confirmed by Socket.IO maintainer](https://github.com/socketio/socket.io/discussions/4754)).
6. **CDP tunneled, not direct** — Chrome never exposed to internet. Fully authenticated. Eliminates Envoy Gateway dependency. Slightly slower but LLM thinking time dwarfs CDP latency.
7. **Logs and metrics forwarded through relay to Datadog** — customer cluster doesn't need direct Datadog access. Relay batches and forwards via Datadog HTTP Intake APIs. Single `DD_API_KEY` on the relay for all customers.
8. **Same API_KEY auth** — no new auth mechanism. Key sent during WS handshake.
9. **X-Client-ID header for routing** — relay routes by `ClientInstallation.id`, not company/team/user. The admin service (`archie-service-admin`) resolves entity → `ClientInstallation` and returns the `client_id`. The relay is a dumb router keyed on `client_id` only. If the SDK's lookup key changes (e.g., from `company_id` to `team_id`), only the SDK and admin service change — the relay never sees it.
10. **Redis for connection registry** — required for multi-replica relay. Socket.IO Redis adapter handles cross-replica routing automatically.
11. **Flask routes kept for local dev** — `MAIN_SERVER_URL` not set or relay unreachable → HTTP mode. Telepresence workflow unchanged.
12. **Envoy Gateway stays** — serves its own purpose independently of the relay. Only dynamic CDP route creation code is removed (CDP goes through tunnel).
13. **Local Chrome for daily dev** — existing fallback behavior. docker-compose only for full end-to-end integration testing.
14. (moved)
15. (moved)
16. **Path-prefix routing via GKE Gateway** — relay is just another backend behind `os.api-k.blitzy.dev`, using `/v1/relay` path prefix. Same pattern as chat service (`/v1/api/chat`). Mediator only knows `MAIN_SERVER_URL` — no separate relay URL.
17. **Unified SDK (`BlitzyClient`)** — all relay-bound traffic (consumers, admin service, future services) uses the same `BlitzyClient` from `blitzy-utils-python`. Admin's internal `HttpForwardingClient` is removed. One SDK, one relay communication pattern.
18. **Relay URL is infrastructure, not per-installation** — the relay URL is not stored in `ClientInstallation.configuration`. It's a global infrastructure endpoint behind the GKE Gateway. Mediator doesn't advertise it.
14. **Redis pub/sub for cross-replica routing** — Socket.IO Redis adapter uses `PUBLISH`/`SUBSCRIBE` (not queues) for real-time cross-pod message delivery. Any relay pod can handle any HTTP request; Redis routes to the pod holding the mediator's socket. ~0.1-0.5ms additional latency.
15. **Relay rolling updates with zero downtime** — `maxUnavailable: 0`, `preStop` sleep 30s, Socket.IO auto-reconnect with infinite retries. Mediator re-registers on every connect. ~1-2s reconnect window per mediator, invisible to consumer jobs (LLM thinking time dwarfs it).
16. **Piece-by-piece local testing** — relay stays on cluster, reachable via Telepresence or `kubectl port-forward`. Run only the component under test locally. No need to recreate the full pipeline.

---

## Verification

1. **Relay health**: `GET /health/connections` shows connected mediators
2. **Control plane**: SDK creates runner → relay → WS → mediator → K8s deployment created
3. **Command execution**: SDK submits command → relay → WS → mediator → Redis → worker executes → result flows back
4. **CDP tunnel**: `chrome-devtools-mcp` → local proxy → relay → mediator → worker Chrome → screenshot returned
5. **Multi-tenant**: Two mediators connected (dedicated + default), SDK requests routed to correct one by `X-Client-ID`
6. **Mediator offline**: SDK gets 503 from relay when mediator disconnected
7. **Reconnect**: Kill mediator, restart → Socket.IO reconnects, registry updated, requests flow again
8. **Log forwarding**: Mediator emits logs on `/logs` namespace → relay batches → Datadog Logs Intake receives them → visible in Datadog log explorer with `company_id` tag
9. **Metrics forwarding**: Mediator emits metrics on `/metrics` namespace → relay forwards → Datadog Metrics Intake receives them → `blitzy.runner.*` metrics visible in Datadog dashboards
10. **Local dev (Mode A)**: `MAIN_SERVER_URL` unset or relay unreachable → Flask HTTP mode, Telepresence works, local Chrome works
11. **Piece-by-piece testing (Mode B)**: TunnelProxy / SDK / mediator WS client runs locally, connects to cluster relay via Telepresence or port-forward
12. **Full integration test (Mode C)**: docker-compose up → full WS flow works end-to-end, no cluster dependency
13. **Envoy Gateway**: Stays active, serves its own purpose. Only CDP route creation removed — CDP goes through tunnel.
14. **Multi-replica routing**: SDK HTTP (with `X-Client-ID`) → any relay pod → Redis pub/sub → correct pod → mediator WS → ack → response. No sticky sessions needed.
15. **Rolling update resilience**: Kill relay pod → mediator reconnects in ~1-2s → re-registers → requests resume. In-flight requests get 503/504 → SDK retries.
16. **Gateway path routing**: `os.api-k.blitzy.dev/v1/relay/*` routes to relay service, URL prefix stripped. Mediator connects WS to `MAIN_SERVER_URL/v1/relay/socket.io` — no new env var.
17. **Admin secret push**: Admin service uses `BlitzyClient` → relay (internal K8s svc) → WS → mediator receives secrets. `HttpForwardingClient` removed.
18. **Unified SDK**: Both consumer jobs and admin service use `BlitzyClient` from `blitzy-utils-python` to reach mediators through relay.
