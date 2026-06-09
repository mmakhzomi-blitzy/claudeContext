# WebSocket Migration — Agent Action Plan

## Context

Reference: `WEBSOCKET_MIGRATION_PLAN.md` contains the full architectural plan.

**Problem**: The mediator currently exposes a public HTTP API via Envoy Gateway. Chrome CDP routes are exposed publicly with zero authentication — a security risk for enterprise client data.

**Solution**: Replace all external communication with a single outbound WebSocket connection from the mediator to a new WS Relay Service on Blitzy K8s. The mediator becomes a purely outbound service — no public endpoints, no Envoy Gateway. Chrome CDP is tunneled through the same WebSocket, fully authenticated.

**Protocol**: Socket.IO (via `python-socketio` / `Flask-SocketIO`) over HTTP/1.1 WebSocket upgrade. Application-level multiplexing via Socket.IO namespaces (`/control`, `/cdp`, `/logs`, `/metrics`). Default 25s ping/pong heartbeat.

**Routing**: Mediator connects to `MAIN_SERVER_URL/v1/relay/socket.io` (no new env var). GKE Gateway at `os.api-k.blitzy.dev` routes `/v1/relay/*` to relay service via path-prefix matching (same pattern as `/v1/api/chat` for chat service). SDK sends HTTP to same gateway with `X-Client-ID` header for routing.

**Testability**: No tests are being written as part of this plan. However, all code MUST be written with dependency injection and interface-based design so that fakes/mocks can be injected at any time. Every service class must depend on abstractions (ABC interfaces), not concrete implementations. Follow the same pattern used in `archie-client-mediator` — see `src/services/interfaces/` and `src/utils/interfaces/` for reference.

---

## Repository 1: `archie-service-relay` (Blitzy K8s)

### Current State
- Blank Flask template deployed on Cloud Run (Knative `service.yaml`)
- Has: Flask app, CORS, health checks (`/v1/health-check`, `/v1/uptime-check`), error handling, Gunicorn, `blitzy-utils`, `blitzy-flask-utils`
- Does NOT have: Socket.IO, Redis, relay logic, namespaces, connection registry

### Target State
A Socket.IO server + transparent HTTP proxy that bridges HTTP callers to WebSocket-connected mediators. Designed for Kubernetes deployment behind the existing GKE Gateway.

### Target Directory Structure

Follow the same layout as `archie-client-mediator`. All interactions are interface-based (ABC) for testability — fakes/mocks can be injected at any time.

```
archie-service-relay/
├── main.py                                    # Flask + SocketIO init (rewrite existing)
├── requirements.txt                           # UPDATE — add socketio, gevent, redis
├── Dockerfile                                 # UPDATE — gevent worker class
├── src/
│   ├── consts.py                              # UPDATE — add Redis config (host/port/username/password)
│   ├── api/
│   │   ├── models.py                          # NEW — request/response Pydantic models
│   │   └── routes/
│   │       ├── __init__.py                    # EXISTS
│   │       ├── utils.py                       # EXISTS — health checks (keep as-is)
│   │       ├── relay.py                       # NEW — HTTP→WS proxy (X-Client-ID routing)
│   │       └── connections.py                 # NEW — GET /health/connections
│   ├── error/
│   │   ├── __init__.py                        # EXISTS
│   │   ├── base_error.py                      # EXISTS — keep
│   │   └── errors.py                          # EXISTS — add relay-specific errors
│   ├── services/
│   │   ├── __init__.py                        # NEW
│   │   ├── interfaces/
│   │   │   ├── __init__.py                    # NEW
│   │   │   ├── i_auth_service.py              # NEW — ABC for auth
│   │   │   └── i_registry_service.py          # NEW — ABC for connection registry
│   │   ├── auth_service.py                    # NEW — API key validation
│   │   └── registry_service.py                # NEW — Redis connection registry
│   ├── namespaces/
│   │   ├── __init__.py                        # NEW
│   │   ├── control.py                         # NEW — /control namespace
│   │   └── tunnel.py                          # NEW — /tunnel namespace
│   ├── utils/
│   │   ├── __init__.py                        # EXISTS
│   │   ├── interfaces/
│   │   │   ├── __init__.py                    # NEW
│   │   │   └── i_redis_client_provider.py     # NEW — ABC for Redis client
│   │   └── redis_client_provider.py           # NEW — Redis client provider
│   ├── models_config/                         # EXISTS — keep as-is
│   └── service/
│       └── base_service.py                    # EXISTS — keep as-is
├── swagger.yaml                               # EXISTS
└── service.yaml → K8s Deployment + Service    # REPLACE — Knative → K8s manifests
```

### Important Conventions

- **Models**: All API models (`src/api/models.py`) are auto-generated from `swagger.yaml` using `make pre-setup`. Do NOT write models manually — define schemas in `swagger.yaml` and run `make pre-setup` to generate them.
- **Kubernetes**: This service is designed to be deployed on Kubernetes. The Knative `service.yaml` will be replaced with K8s Deployment + Service manifests.

### Changes

#### New dependencies (`requirements.txt`)
Add to existing:
```
flask-socketio>=5.3.0
gevent>=24.0.0
gevent-websocket>=0.10.1
redis>=5.0.0
```

#### Rewrite `main.py`
- Initialize `Flask` + `SocketIO` with Redis message queue (construct URL from `REDIS_HOST`, `REDIS_PORT`, `REDIS_USERNAME`, `REDIS_PASSWORD`) and gevent async mode
- `max_http_buffer_size=10 * 1024 * 1024` (10 MB for CDP screenshots)
- Register existing `utils_bp` blueprint (health checks)
- Register new blueprints: `relay_bp`, `connections_bp`
- Register Socket.IO namespaces: `ControlNamespace('/control')`, `TunnelNamespace('/tunnel')`
- Wire up dependencies: create concrete services, inject into namespaces and routes
- Keep existing error handlers and middleware

#### New file: `src/services/interfaces/i_auth_service.py`
- `IAuthService(ABC)` with `authenticate(api_key: str) -> str` abstract method

#### New file: `src/services/auth_service.py`
- `AuthService(IAuthService)` — validates API key, returns `client_id` (`ClientInstallation.id`)
- Constructor takes dependencies needed for validation (e.g., HTTP client or Redis client)
- Implementation options: call admin service `GET /v1/client/auth/validate`, or JWT decode, or Redis cache lookup

#### New file: `src/services/interfaces/i_registry_service.py`
- `IRegistryService(ABC)` with abstract methods: `register(client_id, sid)`, `unregister(client_id)`, `is_online(client_id) -> bool`, `get_sid(client_id) -> str`, `get_all() -> list`

#### New file: `src/services/registry_service.py`
- `RegistryService(IRegistryService)` — Redis CRUD for connection state
- Constructor takes `IRedisClientProvider` (injected)
- Redis key pattern: `mediator:{client_id}` → `{sid, status, connected_at, last_heartbeat}`

#### New file: `src/utils/interfaces/i_redis_client_provider.py`
- `IRedisClientProvider(ABC)` with `get_redis_client() -> redis.Redis` abstract method

#### New file: `src/utils/redis_client_provider.py`
- `RedisClientProvider(IRedisClientProvider)` — creates Redis client from `REDIS_HOST`, `REDIS_PORT`, `REDIS_USERNAME`, `REDIS_PASSWORD`

#### New file: `src/api/routes/relay.py`
- Transparent HTTP→WS proxy: catches any path with `X-Client-ID` header
- `@relay_bp.route('/<path:path>')` for GET/POST/PUT/DELETE
- Wraps request into `{method, path, body, query}` payload
- Uses `IRegistryService` to check if mediator is online, get `sid` for routing
- Calls `socketio.call('request', payload, room=client_id, namespace='/control', timeout=30)`
- Returns 400 if missing `X-Client-ID`, 503 if mediator offline

#### New file: `src/api/routes/connections.py`
- `GET /health/connections` — returns all connected mediators and their status
- Uses `IRegistryService.get_all()` for data

#### New file: `src/namespaces/control.py`
- `ControlNamespace(Namespace)` — handles mediator registration and request routing
- Constructor takes `IAuthService` and `IRegistryService` (injected)
- `on_register(data)` — calls `IAuthService.authenticate()`, `join_room(client_id)`, calls `IRegistryService.register()`
- `on_disconnect()` — calls `IRegistryService.unregister()`

#### New file: `src/namespaces/tunnel.py`
- `TunnelNamespace(Namespace)` — generic port forwarding (stateless relay)
- `on_tunnel_data(data)` — forwards frame to the other side by `runner_id` room
- `on_tunnel_open(data)` / `on_tunnel_close(data)` — room join/leave for tunnel sessions

#### Update `src/consts.py`
Add Redis configuration from environment variables (same pattern as `archie-client-mediator`):
```python
REDIS_HOST = os.getenv("REDIS_HOST", "localhost")
REDIS_PORT = int(os.getenv("REDIS_PORT", "6379"))
REDIS_USERNAME = os.getenv("REDIS_USERNAME")
REDIS_PASSWORD = os.getenv("REDIS_PASSWORD")
```
- Keep existing: `SERVICE_NAME`, `PORT`, `LOG_LEVEL`, `PROJECT_ID`

#### Update `Dockerfile`
Change entrypoint from threaded Gunicorn to gevent WebSocket worker:
```dockerfile
# FROM: gunicorn --bind :$PORT --workers 1 --threads 8 --timeout 0 main:app
# TO:
CMD exec gunicorn --bind :$PORT \
    --worker-class geventwebsocket.gunicorn.workers.GeventWebSocketWorker \
    --workers 1 \
    --timeout 0 \
    main:app
```

#### Replace `service.yaml` (Knative) with K8s Deployment + Service
- Deployment: 2 replicas, `maxUnavailable: 0`, `maxSurge: 1`, `terminationGracePeriodSeconds: 60`
- `preStop` hook: `sleep 30` for connection draining
- Service: ClusterIP on port 8080
- Liveness/readiness probes on `/v1/health-check`

### Final Outcome
- Relay accepts WebSocket connections from mediators on `/socket.io` endpoint
- Relay accepts HTTP requests from SDK/admin with `X-Client-ID` header, forwards to correct mediator via WS
- Multi-replica support via Socket.IO Redis adapter (pub/sub)
- `GET /health/connections` shows all connected mediators
- Returns 503 when target mediator is offline
- All services are interface-based — injectable with fakes/mocks for testing

---

## Repository 2: `archie-client-mediator` (Customer K8s)

### Current State
- Flask app with REST API routes (runners, commands, secrets, registration, utils)
- RQ worker subprocess for async job processing
- Services: `RunnerService`, `CommandService`, `KubernetesService`, `BlitzyService`, `SecretsService`, `RedisQueueService`
- `BlitzyService` uses `HttpClient` for outbound calls to admin service
- `KubernetesService` creates Chrome CDP routes via Gateway API (HTTPRoute CRD + K8s Service)
- `RunnerService` calls `create_chrome_route()`/`delete_chrome_route()` during runner lifecycle
- Env vars: `MAIN_SERVER_URL`, `BLITZY_CLIENT_URL`, `API_KEY`, `GATEWAY_HOST`, `GATEWAY_NAME`

### Target State
Mediator connects outbound to relay via Socket.IO. Receives relayed requests, dispatches to existing service layer. CDP tunneled through WebSocket. Envoy Gateway stays as-is (serves its own purpose). Flask routes remain active alongside WS client.

### Prerequisite: Chrome CDP Routing

These changes add Chrome CDP routing capability to the mediator. They must be applied **before** the WebSocket migration changes below.

**Files to modify (8 files):**

#### `swagger.yaml` — add capability schema + fields
- Add `RunnerCapability` enum schema:
  ```yaml
  RunnerCapability:
    type: string
    enum: [CHROME]
    description: Worker capability that can be enabled on demand
  ```
- Add to `CreateRunnerConfig`:
  ```yaml
  capabilities:
    type: array
    items:
      $ref: '#/components/schemas/RunnerCapability'
    description: Optional list of capabilities to enable on the worker
    example: [CHROME]
  ```
- Add to `CreateRunnerResponse` and `RunnerStatusResponse`:
  ```yaml
  capabilities:
    type: array
    items:
      $ref: '#/components/schemas/RunnerCapability'
    description: Capabilities enabled on this runner
  capabilities_metadata:
    type: object
    additionalProperties: true
    description: Capability-specific metadata (e.g., chrome_url for CHROME capability)
  ```

#### `src/consts.py` — add gateway constants
```python
# Gateway Configuration
GATEWAY_HOST = os.getenv("GATEWAY_HOST", "")
"""External gateway hostname for capability routing (e.g. Chrome CDP)."""

GATEWAY_NAME = os.getenv("GATEWAY_NAME", "blitzy")
"""Name of the Envoy Gateway resource for HTTPRoute parentRefs."""
```

#### `src/utils/interfaces/i_kube_client_provider.py` — add abstract method
```python
@abstractmethod
def get_custom_objects_api(self) -> client.CustomObjectsApi:
    """Return CustomObjectsApi for CRD operations (e.g. HTTPRoute)."""
    pass
```

#### `src/utils/kube_client_provider.py` — add implementation
```python
def get_custom_objects_api(self) -> client.CustomObjectsApi:
    """Return CustomObjectsApi for CRD operations (HTTPRoute)."""
    return client.CustomObjectsApi()
```

#### `src/services/interfaces/i_kubernetes_service.py` — add abstract methods
```python
@abstractmethod
def create_chrome_route(self, runner_id: str, namespace: str) -> None:
    """Create K8s Service + HTTPRoute for Chrome CDP access."""
    pass

@abstractmethod
def delete_chrome_route(self, runner_id: str, namespace: str) -> None:
    """Delete Chrome CDP K8s Service + HTTPRoute. Idempotent."""
    pass
```

#### `src/services/kubernetes_service.py` — major additions
- Add `self._custom_api: Optional[client.CustomObjectsApi] = None` to `__init__`
- Add `_get_custom_api()` lazy loader (same pattern as `_get_apps_api()`)
- Add `_build_chrome_service_name(runner_id)` → `f"chrome-svc-{sanitized_id}"`
- Add `_build_chrome_route_name(runner_id)` → `f"chrome-cdp-{sanitized_id}"`
- Add `create_chrome_route(runner_id, namespace)`:
  - Creates ClusterIP Service on port 9222 targeting worker pod (uses `_build_labels()` for selector)
  - Creates HTTPRoute CRD via `CustomObjectsApi` with path prefix `/cdp/{runner_id}`, parentRef to `GATEWAY_NAME` (sectionName: `https`)
  - On HTTPRoute failure: cleans up the Service before raising
  - Idempotent: ignores 409 conflict on both resources
- Add `delete_chrome_route(runner_id, namespace)`:
  - Deletes HTTPRoute via `CustomObjectsApi` (group: `gateway.networking.k8s.io`, version: `v1`, plural: `httproutes`)
  - Deletes ClusterIP Service
  - Idempotent: ignores 404 on both resources
- Add `_delete_k8s_service(service_name, namespace)` helper (idempotent, ignores 404)
- Import `GATEWAY_NAME` from `src.consts`
- In `create_deployment()`: inject `RUNNER_CAPABILITIES` env var (JSON array) when `config.capabilities` is present

#### `src/services/runner_service.py` — capability lifecycle integration
- Import `RunnerCapability` from `src.api.models` and `GATEWAY_HOST` from `src.consts`
- In `create_runner()` — insert new Step 2 (before Redis queue provisioning):
  ```python
  capabilities = list(getattr(config, "capabilities", None) or [])
  capabilities_metadata = {}
  if RunnerCapability.CHROME in capabilities:
      self._kubernetes_service.create_chrome_route(runner_id, namespace)
      if GATEWAY_HOST:
          capabilities_metadata["chrome_url"] = f"https://{GATEWAY_HOST}/cdp/{runner_id}"
  ```
- In `create_runner()`: pass `capabilities` and `capabilities_metadata` to `_build_server_metadata()`
- `_build_server_metadata()` — add `capabilities` and `capabilities_metadata` params, store in metadata dict
- In `delete_runner()`: read `server_capabilities` from `job_server.server_metadata`, add Step 6 to delete chrome route if `CHROME` in capabilities
- `_cleanup_failed_creation()` — rename param from `blitzy_job_id` to `runner_id`, add `capabilities` param, add Chrome route cleanup block at the top

#### `src/api/routes/runners.py` — expose capabilities in responses
- In `create_runner()`: extract `server_metadata` from `job_server`, pass `capabilities` and `capabilities_metadata` to `CreateRunnerResponse`
- In `get_runner_status()`: extract `server_metadata` from `job_server`, pass `capabilities` and `capabilities_metadata` to `RunnerStatusResponse`

---

### WebSocket Migration Changes

#### New dependency (`requirements.txt`)
Add: `python-socketio[asyncio]>=5.0.0`

#### New file: `src/services/ws_client.py`
- `WSClient` class — outbound Socket.IO connection to relay
- Constructor: `relay_url`, `api_key`, creates `socketio.Client` with infinite reconnect (`reconnection_attempts=0`, delay 1-5s)
- `_register_handlers()` — registers `on('request')` on `/control`, `on('connect')` emits `register` with API key
- `start()` — connects to relay with namespaces `['/control', '/tunnel']`
- `stop()` — disconnects
- On every `connect` event (including reconnect): re-registers by emitting `{'api_key': API_KEY}`

#### New file: `src/services/ws_router.py`
- `WSRouter` class — dispatches WS messages to existing service methods
- `dispatch(message: dict) -> dict` — routes by `method` + `path`
- Route mapping:
  - `POST /api/v1/runners` → `RunnerService.create_runner()`
  - `GET /api/v1/runners/{id}` → `RunnerService.get_runner()` / status
  - `DELETE /api/v1/runners/{id}` → `RunnerService.delete_runner()`
  - `POST /api/v1/jobs/{id}/commands` → `CommandService.submit_command()`
  - `GET /api/v1/commands/{id}/status` → `CommandService.get_command_status()`
  - `POST /api/v1/secrets/all` → `SecretsService` (secret push from admin)
  - `POST /api/v1/jobs/{id}/restart-session` → `CommandService.restart_session()`
- Returns dict response (same shape as Flask JSON responses)
- Error handling: wraps service exceptions into `{error, status_code}` dicts

#### New file: `src/services/tunnel_forwarder.py`
- `TunnelForwarder` class — forwards tunnel frames between relay and worker K8s services
- Manages TCP connections to worker pods (e.g., Chrome on `:9222`)
- `handle_open(runner_id, port, connection_id)` — opens `asyncio.open_connection` to `chrome-svc-{runner_id}.{namespace}.svc.cluster.local:{port}`
- `handle_data(runner_id, port, connection_id, data)` — forwards data to worker
- `handle_close(runner_id, port, connection_id)` — closes connection
- Background read loop: reads from worker TCP, emits back to relay via Socket.IO `/tunnel` namespace

#### Modify `main.py`
After existing Flask setup and before worker startup, add conditional WS client:
```python
if MAIN_SERVER_URL:
    relay_url = f"{MAIN_SERVER_URL}/v1/relay"
    from src.services.ws_client import WSClient
    ws_client = WSClient(relay_url=relay_url, api_key=API_KEY)
    ws_client.start()
```
- Flask routes, RQ worker, health checks all stay unchanged
- WS client runs alongside them

#### Modify `src/consts.py`
- Keep all existing: `MAIN_SERVER_URL`, `API_KEY`, `BLITZY_CLIENT_URL`, `GATEWAY_HOST`, `GATEWAY_NAME` (Envoy Gateway stays)
- No new env vars needed

#### Modify `src/services/blitzy_service.py`
- `register()` method: when WS client is active, skip sending URL (mediator has no public URL). Registration becomes implicit on WS connect. Keep HTTP fallback for local dev.
- `get_job_info()`, `register_job()`, `request_secret_sync()`: **no change** — these are outbound HTTP calls to admin service, they stay as direct HTTP via `HttpClient`

#### Modify `src/services/runner_service.py` (Phase 4)
- `create_runner()`: Remove `self._kubernetes_service.create_chrome_route(runner_id, namespace)` call and `capabilities_metadata["chrome_url"]` URL construction. Keep `create_service()` for ClusterIP (needed for tunnel).
- `delete_runner()`: Remove `self._kubernetes_service.delete_chrome_route(runner_id, namespace)` call
- `_cleanup_failed_creation()`: Remove chrome route cleanup block

#### Modify `src/services/kubernetes_service.py` (Phase 4)
- Remove methods: `create_chrome_route()`, `delete_chrome_route()`, `_build_chrome_service_name()`, `_build_chrome_route_name()`, `_delete_k8s_service()` (if only used for CDP)
- Keep: `create_deployment()`, `delete_deployment()`, `get_deployment_status()`, `create_service()`, `_get_custom_api()`, `CustomObjectsApi`

#### Modify `src/services/interfaces/i_kubernetes_service.py` (Phase 4)
- Remove abstract methods: `create_chrome_route()`, `delete_chrome_route()`

#### No changes to (Envoy Gateway stays)
- `src/utils/interfaces/i_kube_client_provider.py` — `get_custom_objects_api()` stays
- `src/utils/kube_client_provider.py` — `get_custom_objects_api()` stays
- `BLITZY_CLIENT_URL`, `GATEWAY_HOST`, `GATEWAY_NAME` in `src/consts.py` — stay

### Final Outcome
- Mediator connects outbound to relay at `MAIN_SERVER_URL/v1/relay/socket.io`
- Receives relayed SDK/admin requests on `/control` namespace, dispatches to existing services
- Tunnel frames on `/tunnel` namespace forwarded to worker Chrome/services
- Auto-reconnect with infinite retries (1-5s backoff)
- Flask routes and Envoy Gateway stay active (serve their own purpose)
- Flask health probes still work for K8s liveness/readiness
- CDP route creation removed (CDP goes through tunnel), but gateway itself stays

---

## Repository 3: `blitzy-utils-python` (Shared SDK)

### Current State
- `BlitzyClient` in `blitzy_utils/blitzy_utils/blitzy_client.py`
- `_RequestsAdapter`: simple pass-through to `requests.get/post/delete`
- `get_client_url(company_id)`: calls admin API `GET /v1/client/entity/COMPANY/{company_id}`, caches `configuration.url` in `self._server_url`
- All methods follow pattern: `get_client_url()` → build URL → `self._http_client.post/get/delete(url, ...)` → return JSON
- `BlitzyHttpClient` Protocol for dependency injection

### Target State
`BlitzyClient` becomes the unified SDK for all relay-bound communication (consumers + admin service + future services). Injects `X-Client-ID` header on every request for relay routing.

### Changes

#### Modify `blitzy_utils/blitzy_utils/blitzy_client.py`

**`_RequestsAdapter` class** — add `X-Client-ID` header injection:
```python
class _RequestsAdapter:
    def __init__(self):
        self._client_id: str | None = None

    def _with_client_id(self, kwargs: dict) -> dict:
        if self._client_id:
            headers = kwargs.get("headers") or {}
            headers["X-Client-ID"] = self._client_id
            kwargs["headers"] = headers
        return kwargs

    def get(self, url, **kwargs):
        return requests.get(url, **self._with_client_id(kwargs))
    def post(self, url, **kwargs):
        return requests.post(url, **self._with_client_id(kwargs))
    def delete(self, url, **kwargs):
        return requests.delete(url, **self._with_client_id(kwargs))
```

**`BlitzyClient.__init__`** — add `_client_id` field:
```python
self._client_id: Optional[str] = None
```

**`BlitzyClient.get_client_url()`** — cache `client_id` from admin response:
```python
self._client_id = configuration.get("client_id")
if self._client_id and isinstance(self._http_client, _RequestsAdapter):
    self._http_client._client_id = self._client_id
```

**Backwards compatible**: When `client_id` is `None` (admin not yet updated), no header injected. SDK works exactly as before.

**No changes to**: `BlitzyHttpClient` Protocol, method signatures, request/response payloads, `RunnerSession`, `wait_for_ready`, `execute_command`, `ServiceClient`.

### Final Outcome
- All HTTP requests from `BlitzyClient` include `X-Client-ID` header when `client_id` is available
- Admin service response `configuration.client_id` is cached and injected automatically
- Zero breaking changes — existing consumers work without modification
- Admin service can use same `BlitzyClient` to push secrets through relay

---

## Repository 4: `archie-service-admin` (Blitzy K8s)

### Current State
- `HttpForwardingClient` in `src/clients/http_forwarding_client.py` (270 lines) — pushes secrets to mediator at `configuration.url + /api/v1/secrets/all` with retry logic, URL validation, exponential backoff
- `EnvExchangeService._forward_secrets_to_client()` in `src/service/env_exchange_service.py` (lines 265-286) — calls `self.http_client.forward_secrets(context.forwarded_to_url, payload)`
- `GET /v1/client/entity/{type}/{id}` — returns `ClientInstallationModel` with `client_id` at top level and `configuration` dict

### Target State
Admin service uses `BlitzyClient` (from `blitzy-utils-python`) for secret push through relay, replacing `HttpForwardingClient`. Internal K8s service URL to relay (no gateway needed).

### Changes

#### Add dependency (`requirements.txt`)
Add: `blitzy-utils>=0.0.38` (for `BlitzyClient`)

#### Modify `src/service/env_exchange_service.py`
Replace `HttpForwardingClient` with `BlitzyClient`:
```python
from blitzy_utils.blitzy_client import BlitzyClient

class EnvExchangeService:
    def __init__(self, blitzy_client=None):
        self._blitzy_client = blitzy_client or BlitzyClient()

    def _forward_secrets_to_client(self, context, gsm_secrets):
        massaged_payload = self._build_forward_secret_payload(gsm_secrets)
        client_id = str(context.client.id)
        self._blitzy_client.push_to_mediator(
            client_id=client_id,
            path="/api/v1/secrets/all",
            payload=massaged_payload,
        )
```

`BlitzyClient` may need a `push_to_mediator(client_id, path, payload)` method for direct relay calls without `get_client_url()` resolution. This is because admin service already knows the `client_id` — no need to look it up.

#### Remove `src/clients/http_forwarding_client.py` (Phase 4)
Entire file removed — replaced by `BlitzyClient`.

#### No changes to
- `GET /v1/client/entity/{type}/{id}` — already returns `client_id`
- `PUT /v1/client/register` — mediator still calls for heartbeat
- API key auth middleware
- Client models, repository layer
- Default installation logic (`get_default_client_for_entity()`)

### Final Outcome
- Secret push routes through relay: Admin → `BlitzyClient` → relay (internal K8s svc) → WS → mediator
- `HttpForwardingClient` removed (retry logic now handled by `BlitzyClient` or relay)
- Unified SDK pattern: same `BlitzyClient` used by consumers and admin service

---

## Repository 5: `archie-helm-chart` (Infrastructure)

### Current State

**`api-gateway/` chart:**
- GKE Gateway at `platform.api-k.blitzy.dev` with `gke-l7-global-external-managed`
- Single HTTPRoute rule: `/v1/api/chat` → `archieservicechat-dev:8080` (URL prefix stripped)
- `GCPBackendPolicy`: `timeoutSec: 3600` (1 hour for SSE)
- Templates: `gateway.yaml`, `route.yaml`, `backend-policy.yaml`, `health-check.yaml`

**`blitzy-client/chart/` chart:**
- Mediator deployment + ClusterIP Service
- Envoy Gateway: `gateway.enabled: true`, class `blitzy-envoy-gateway`, domain `chaitanya-client.blitzy.dev`
- ConfigMap with: `MAIN_SERVER_URL: https://os.api-k.blitzy.dev`, `BLITZY_CLIENT_URL: https://chaitanya-client.blitzy.dev`, `API_KEY`, etc.
- Dependencies: cert-manager, gateway-helm v1.6.2, redis, cloudnative-pg, vault, otel

### Target State
- New `/v1/relay` path-prefix route on GKE Gateway pointing to relay service
- GCPBackendPolicy with 24-hour timeout for WebSocket
- Envoy Gateway on blitzy-client chart stays as-is (serves its own purpose)
- New relay service Helm chart for K8s deployment

### Changes

#### `api-gateway/values.yaml` — add relay routing rule
Add to existing `rules` array:
```yaml
- matches:
  - path:
      type: PathPrefix
      value: /v1/relay
  backendRefs:
  - kind: Service
    name: archie-service-relay
    port: 8080
  filters:
  - type: URLRewrite
    urlRewrite:
      path:
        replacePrefixMatch: /
        type: ReplacePrefixMatch
```

Add backend policy for relay (24-hour WebSocket timeout):
```yaml
relayBackendPolicy:
  enabled: true
  timeoutSec: 86400
  drainingTimeoutSec: 30
  targetService: archie-service-relay
```

#### New: Relay service Helm chart or K8s manifests
- Deployment: 2 replicas, rolling update (`maxUnavailable: 0`, `maxSurge: 1`)
- Service: ClusterIP on port 8080
- `terminationGracePeriodSeconds: 60`, `preStop` sleep 30s
- Env: `REDIS_HOST`, `REDIS_PORT`, `REDIS_USERNAME`, `REDIS_PASSWORD`, `PORT`, `LOG_LEVEL`
- Resources: 100m-1000m CPU, 256Mi-1Gi memory
- Probes: HTTP GET `/v1/health-check` on port 8080

#### `blitzy-client/chart/` — No changes
Envoy Gateway stays as-is. All existing config (`BLITZY_CLIENT_URL`, `GATEWAY_HOST`, `GATEWAY_NAME`, gateway-helm dependency, templates) remains unchanged.

### Final Outcome
- `os.api-k.blitzy.dev/v1/relay/*` routes to relay service (prefix stripped)
- 24-hour WebSocket timeout via GCPBackendPolicy
- Envoy Gateway on blitzy-client chart stays active (serves its own purpose)
- No changes to mediator Helm config — `MAIN_SERVER_URL` already exists

---

## Repository 6: `blitzy-utils-python` — Consumer CDP Proxy

### Current State
- `runner_ops/` directory contains runner operation modules
- Consumers use `chrome-devtools-mcp` with direct `--browserUrl` to exposed Chrome

### Target State
Local CDP proxy on consumer side tunnels Chrome frames through relay to mediator.

### Changes

#### New file: `blitzy_utils/blitzy_utils/runner_ops/cdp_proxy.py`
- `CDPProxy` class — local proxy on `localhost:9222`
- HTTP discovery: tunnels `GET /json/version`, `GET /json/list` through relay
- WebSocket: tunnels `/devtools/browser/...` frames bidirectionally through relay `/tunnel` namespace
- Library: `websockets`

#### Modify consumer helper (e.g., `archie-job-reverse-code-generator/lib/blitzy/helper.py`)
- When `runner_session.chrome_url` exists: start `CDPProxy`, configure `chrome-devtools-mcp` to use `localhost:9222`
- When no chrome capability: use local Chrome (existing fallback)

### Final Outcome
- `chrome-devtools-mcp` connects to `localhost:9222` (local proxy)
- Frames tunnel through relay → mediator → worker Chrome
- Chrome never exposed to internet
- Screenshots, navigation, evaluation all work through tunnel

---

## Repository 7: `archie-client-worker` (Customer K8s)

### Current State
- Worker pod runs bash sessions and processes commands from Redis queue
- No Chrome support — Chrome is not installed in the Docker image
- Startup pipeline uses builder pattern: `WorkerStartupBuilder` with `StartupStep` classes
- `RUNNER_CAPABILITIES` env var support does not exist yet

### Target State
Worker supports Chrome headless for remote debugging (CDP on port 9222), activated dynamically via `RUNNER_CAPABILITIES` environment variable. Chrome is installed in the Docker image but only started when the `CHROME` capability is requested.

### Changes

#### Modify `Dockerfile` — install Chrome
Add after existing apt-get installs:
```dockerfile
# Google Chrome (for remote debugging — started conditionally via RUNNER_CAPABILITIES)
RUN wget -q -O - https://dl.google.com/linux/linux_signing_key.pub | gpg --dearmor -o /usr/share/keyrings/google-chrome-keyring.gpg && \
    echo "deb [arch=amd64 signed-by=/usr/share/keyrings/google-chrome-keyring.gpg] http://dl.google.com/linux/chrome/deb/ stable main" > /etc/apt/sources.list.d/google-chrome.list && \
    apt-get update && \
    apt-get install -y google-chrome-stable && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* && \
    google-chrome --version

ENV DBUS_SESSION_BUS_ADDRESS=/dev/null \
    CHROME_DEVEL_SANDBOX=0
```

- `DBUS_SESSION_BUS_ADDRESS=/dev/null` — disables D-Bus (required for headless Chrome in containers)
- `CHROME_DEVEL_SANDBOX=0` — disables sandbox (needed for rootless container execution)
- `wget` and `gnupg` must be available (add to apt-get if not already installed)

#### Modify `src/blitzy_worker/startup.py` — add `StartChromeStep`
New startup step class:
```python
class StartChromeStep(StartupStep):
    """Start Chrome headless for remote debugging (CDP on port 9222)."""

    @property
    def name(self) -> str:
        return "start_chrome"

    def execute(self, context: StartupContext) -> None:
        chrome_bin = shutil.which("google-chrome")
        if not chrome_bin:
            logger.warning("chrome_not_found", message="Chrome not installed, skipping")
            return
        subprocess.Popen([
            chrome_bin, "--headless", "--no-sandbox",
            "--remote-debugging-port=9222",
            "--disable-dev-shm-usage", "--disable-gpu",
            "--user-data-dir=/tmp/chrome-profile",
        ])
        logger.info("chrome_started", port=9222)
```

Add capability registry:
```python
CAPABILITY_STEPS: dict[str, type[StartupStep]] = {
    "CHROME": StartChromeStep,
}
```

Chrome flags:
- `--headless` — run without UI
- `--no-sandbox` — required for K8s containers
- `--remote-debugging-port=9222` — expose CDP
- `--disable-dev-shm-usage` — avoids /dev/shm issues in low-memory environments
- `--disable-gpu` — no GPU in containers
- `--user-data-dir=/tmp/chrome-profile` — temporary profile

Graceful degradation: if Chrome binary not found, logs warning and continues without CDP.

#### Modify `main.py` — capability activation
Add capability registration to worker startup:
```python
builder = WorkerStartupBuilder(settings)
for cap in json.loads(os.environ.get("RUNNER_CAPABILITIES", "[]")):
    builder.with_capability(cap)
startup = builder.build()
working_directory = startup.execute()
```

#### Modify `src/blitzy_worker/startup.py` — builder `with_capability` method
```python
def with_capability(self, name: str) -> "WorkerStartupBuilder":
    step_cls = CAPABILITY_STEPS.get(name)
    if step_cls:
        self._steps.append(step_cls())
    else:
        logger.warning("unknown_capability", capability=name)
    return self
```

### Final Outcome
- Chrome installed in worker Docker image (always available)
- Chrome only started when `RUNNER_CAPABILITIES=["CHROME"]` is set
- CDP available on `localhost:9222` inside the worker pod
- Mediator's `TunnelForwarder` connects to `chrome-svc-{runner_id}:9222` to tunnel CDP frames
- Graceful degradation: worker works normally without Chrome if capability not requested
- Pattern is extensible: future capabilities (e.g., dev server) follow the same `StartupStep` + `CAPABILITY_STEPS` pattern

---

## Verification Checklist

1. **Relay health**: `GET /health/connections` shows connected mediators
2. **Control plane**: SDK `create_runner()` → relay → WS → mediator → K8s deployment created
3. **Command execution**: SDK `submit_command()` → relay → WS → mediator → Redis → worker → result flows back
4. **CDP tunnel**: `chrome-devtools-mcp` → local proxy → relay → mediator → worker Chrome → screenshot returned
5. **Multi-tenant**: Two mediators connected, requests routed to correct one by `X-Client-ID`
6. **Mediator offline**: SDK gets 503 from relay
7. **Reconnect**: Kill mediator → Socket.IO reconnects in 1-2s → re-registers → requests resume
8. **Secret push**: Admin → `BlitzyClient` → relay (internal K8s svc) → WS → mediator receives secrets
9. **Local dev**: `MAIN_SERVER_URL` unset → Flask HTTP mode, Telepresence works
10. **Gateway routing**: `os.api-k.blitzy.dev/v1/relay/*` routes correctly, prefix stripped
11. **Envoy Gateway**: Stays active, serves its own purpose independently of the relay
12. **Rolling updates**: Kill relay pod → mediator reconnects → requests resume (1-2s blip)
13. **Worker Chrome**: Worker with `RUNNER_CAPABILITIES=["CHROME"]` → Chrome starts on `:9222` → `curl localhost:9222/json/version` returns browser info
14. **Worker graceful degradation**: Worker without Chrome capability → no Chrome process, worker runs normally
