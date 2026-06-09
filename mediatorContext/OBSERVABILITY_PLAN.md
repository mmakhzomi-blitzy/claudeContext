# Customer Observability Plan — Blitzy Client

> Goal: debug/diagnose blitzy-client deployments in customer ("blackbox") clusters
> without direct kubectl/network access. Capture this session's discussion, the
> hard constraint, and the design space.

---

## 1. The problem

Once blitzy-client is installed in a customer's K8s cluster:

| We lose visibility into | What it means in practice |
|---|---|
| Pod logs (mediator, workers, runners, vault, postgres, redis, otel) | Can't tail/grep during incidents |
| Pod state (Pending/CrashLoopBackOff/OOMKilled/restart counts) | Don't know mediator is being killed every 90s |
| Resource utilization (CPU/mem/disk per pod) | Can't tell load problem vs code bug |
| Redis state (queue depth, stream length, pending entries) | Don't know outbound queue is backing up |
| Postgres state (table sizes, slow queries, lock contention) | Don't know archive table is huge or migrations pending |
| Vault state (sealed/unsealed, raft peers, token TTL) | Don't know auth is about to break |
| K8s API state (Deployment status, NetworkPolicies, admission webhooks) | Don't know customer's policies are blocking us |
| Cert state (cert-manager Order/Challenge, expiry) | Don't know HTTPS is about to break |
| Networking (DNS, egress, proxies, MITM CAs, MTU) | First report is "WebSocket randomly disconnects" |
| Image versions actually running | Customer claims they upgraded; reality says otherwise |
| Customer's `values.yaml` overrides | Don't know what they changed; bug repro fails |
| Helm/Argo state (revision, sync errors) | Don't know if last upgrade actually completed |
| Filesystem on PVCs | Can't inspect crash dumps, hot-data on disk |

## 2. The hard constraint

**No telemetry component in the customer cluster may egress directly to a vendor SaaS** (Datadog, Sentry, NewRelic, etc.). All vendor-bound telemetry must traverse the existing relay path. This is non-negotiable for customer security/compliance.

Implication: the relay is the single egress chokepoint. Anything we want to ship — logs, metrics, traces, exception payloads, support bundles — must go mediator → relay → vendor side, then vendor side fans out to whatever backend.

## 3. What is already in place

These are wins we don't have to redesign:

| Layer | Status | File / location |
|---|---|---|
| **Structured JSON logs** with `image_tag`, `client_id`, `correlation_id` global fields | ✅ done | `blitzy_utils.logger`; `main.py:39`; `ws_client.py:329, 401` |
| **OTel collector DaemonSet** in customer cluster | ✅ done | `chart/values.yaml:188-285` |
| **Filelog receiver** (collects all pod stdout) + **kubeletstats** (node/pod/container metrics) | ✅ done | OTel collector `presets.logsCollection.enabled: true`; explicit `kubeletstats` receiver |
| **OTLP HTTP endpoint on mediator** (`/otlp/...`) — currently handles logs from OTel collector | ✅ done | `src/api/routes/otlp.py` |
| **`/logs` Socket.IO namespace** on mediator → relay — forwards OTLP log batches | ✅ done | mediator `WSClient` connects to `/logs`; relay `namespaces/logs.py:on_logs` forwards to `OTEL_FORWARDER_URL` |
| **Outbound queue store-and-forward** | ✅ done | `STORE_AND_FORWARD.md`; `WSHttpClient` + `mediator:outbound_queue` Redis list |
| **Relay deep health endpoints** (`/api/v1/health/connections`, `/api/v1/health/queue`) — already expose registry, drain stats, queue depth | ✅ done | `relay/src/api/routes/connections.py`, `queue.py` |
| **Pod metadata in OTel** — `blitzy.client_id`, `blitzy.client_name`, `service.name` already tagged on every log/metric | ✅ done | `chart/values.yaml:236-244` |
| **Customer identification** — relay knows `client_id` per WS connection via `auth_service.authenticate()` | ✅ done | relay `auth_service.py:67-94` |
| **Watchdog for RQ worker** | ✅ done (but silent on death) | `main.py:205-217` |
| **`extraManifests:` array** at top of `values.yaml` — escape hatch for shipping arbitrary K8s objects (e.g. SecretProviderClass, ServiceMonitor, PodMonitor) without changing chart templates | ✅ recently added | `chart/values.yaml:1` |
| **Customer secret injection via CSI** — `clientMediator.extraVolumes` / `extraVolumeMounts` | ✅ recently added | `chart/values.yaml:145-146`; `templates/client-mediator.yaml` |

## 4. What is still missing (the gap)

### Gaps in mediator code

| Gap | Severity | Effort |
|---|---|---|
| **Zero metrics emission** — no statsd/prometheus_client/dd-trace anywhere | High | Low (1 day) |
| **No tracing** — no OTel SDK init, no trace propagation across mediator→relay→backend | High | Medium |
| **Deep `/healthz`** — current endpoints are shallow (`/health-check` always returns 200, only `/redis-check` probes) | Medium | Half day |
| **Watchdog cause-of-death** — when watchdog kills parent, no log/metric explaining why | Medium | 30 min |
| **Disconnect events at ERROR severity** — every routine reconnect emits ~10 ERROR lines, polluting alerting | Medium | 30 min |
| **OTLP endpoint silently swallows exceptions** — `routes/otlp.py:102` catches and returns 200 even on parse error | Low | 15 min |
| **`client_id` not preserved through disconnect** — error/warning lines after disconnect have no client_id (cleared on disconnect, repopulated only on register success). Triaging across customers is harder | Medium | 1 hour |
| **No exception telemetry** — Python tracebacks die in stderr; no Sentry-style auto-capture | High | Half day with SaaS, 2-3 days self-hosted |
| **No support-bundle endpoint** — customer can't run a single command to gather diag | Low | 3-5 days |
| **No active heartbeat with rich payload** — relay knows "client connected: yes/no" but not "client healthy: yes/no, here's why" | Medium | Medium |
| **No feature flag layer** — every behavior change requires redeploy | Medium | Medium |
| **No synthetic canary** — no built-in self-test that runs every 5min and emits result | Medium | Medium |
| **No migration visibility** — no admin endpoint showing current alembic revision, pending migrations | Low | Low |
| **No remote diagnostic command channel** — vendor can't ask mediator to "dump redis stream" or "run pg_stat_activity" | High value but big | 1+ week |

### Gaps in relay code

| Gap | Severity | Effort |
|---|---|---|
| **Zero metrics emission** — no statsd/dd-trace anywhere; only logs | High | Low (1 day) |
| **No proactive stale-mediator detection** — only socket drop triggers disconnect; no "client X went silent" heartbeat-loss timer (`PING_TIMEOUT=600s` is passive) | Medium | 2 hours |
| **`_chunk_buffers` leak risk** — incomplete chunked responses linger in memory; no metric, no cleanup timer | Medium | 30 min |
| **No tracing infrastructure** — no SDK, no propagation between mediator and backend | High | Medium |
| **Per-customer queue/drain stats are queryable** (`get_queue_stats()`) but not emitted as metrics | Low | 1 hour |

### Gaps in the chart

| Gap | Severity | Effort |
|---|---|---|
| **Mediator deployment has NO health probes** — neither `livenessProbe` nor `readinessProbe`. K8s can't restart a hung mediator | High | 10 min once `/healthz` exists |
| **No traces pipeline in OTel collector** — only `logs` and `metrics`. Any traces emitted by mediator have nowhere to go | Medium | Half hour |
| **Single Service port (80)** on mediator — no metrics port | Low | 5 min |
| **No `app.kubernetes.io/version` label** — `Chart.yaml` has no `appVersion`, helper template emits empty | Low | 5 min |
| **No autodiscovery annotations** on pods (Datadog AD, Prometheus scrape) | Low | 15 min |
| **OTel collector exports only to mediator `/otlp`** — single point of failure (if relay/mediator stops, customer logs are also lost). Could fan out to local Loki backup OR another customer-controlled destination | Medium | 30 min |

## 5. Architectural options for shipping telemetry off-cluster

Four routes from customer cluster → vendor's observability backend (Datadog, etc.):

| Option | Customer side | Mediator side | Relay side | Trade-off |
|---|---|---|---|---|
| **1. Extend `/logs` to `/telemetry`** | OTel collector adds metrics+traces pipelines, exports OTLP to mediator `/otlp` | Mediator `/otlp` accepts logs/metrics/traces, emits over WS by type | New `/metrics`, `/traces` namespaces (or single `/telemetry` with type field) → forwarder → DD | Most consistent with existing pattern. Custom protocol per-hop. Mediator is in hot path. |
| **2. Direct OTel-over-WS exporter** | Custom OTel exporter that speaks Socket.IO directly to relay | Bypassed entirely | New OTel-receiving namespace | Cleanest data path. Have to write+maintain custom exporter. |
| **3. Relay-side OTel collector** ⭐ | OTel collector exports OTLP/HTTP to mediator (unchanged) | Mediator forwards binary OTLP to relay over WS | **New: relay-side OTel collector** with `datadog`/`loki`/`tempo` exporter + processors (scrubbing, tagging, sampling) | Simplest mental model. Standard OTel everywhere. Vendor side becomes a real OTel pipeline. Switch backends by changing one config. |
| **4. Treat DD as just another backend** | OTel collector unchanged | `WSHttpClient.post("/v1/telemetry/...", queueable=True)` for app-level metrics. OTLP for everything else. | No relay change; backend service forwards to DD | Reuses store-and-forward perfectly. Backend gets coupled to DD. Loses OTel niceties for app metrics. |

### Recommended: Option 3

- All processing (PII scrubbing, sampling, tagging, format conversion) happens in one place we control
- Customer side stays vanilla OTel — no custom exporters to maintain
- Switch DD ↔ Honeycomb ↔ NewRelic by changing one exporter config in our cluster
- Relay-side OTel collector can fan out (DD + Loki + S3 archive in one pipeline)

## 6. Datadog coverage of identified gaps (assuming Option 3)

| Gap | Datadog feature | Setup effort |
|---|---|---|
| `/debug/state` endpoint (queue depths, thread counts) | Not needed — emit DogStatsD gauges; query DD dashboard | Low |
| Support-bundle endpoint | DD Live Tail + DD container live view + APM trace search replaces ~80% | Built-in |
| Sentry-equivalent exception capture | DD Error Tracking — `ddtrace` auto-captures every unhandled exception | One-line code change |
| Active heartbeat from mediator | DD Agent emits per-pod liveness every 15s by default; APM service health | Built-in |
| Feature flags | **Not DD** — need OpenFeature/LaunchDarkly. DD can observe flag state if instrumented | Separate concern |
| `mediator-cli` for diag bundle | DD CLI + API queries reduce need; in-cluster `agent flare` if Agent ships | Built-in |
| Synthetic canary (HTTP) | DD Synthetics for relay endpoint | Built-in |
| Synthetic canary (app-internal: enqueue→worker→archive) | Still need to write logic; emit a custom metric DD watches | Medium |
| Migration visibility | Custom metric: `mediator.alembic.current_revision` gauge tagged with revision | Low |
| Audit log of who triggered what command | Already in `command_executions` table; mirror to DD as Indexed Logs | Low |
| Watchdog cause-of-death | Add `logger.error(reason=...)` before `os.kill(SIGTERM)` + `metrics.incr('watchdog.kill', tags=['reason:rq_exit'])` | 30 min |
| Remote diagnostic command channel | DD Logs Live Tail + APM live search **mostly** replaces this. For active extraction, still need own mechanism | Mostly built-in |
| Pre-flight install check Job | **Not DD** — separate concern | App-level |
| Deep healthcheck | DD Service Check from Agent against `/healthz` | Once we have `/healthz` |

DD also adds free-with-SDK capabilities we didn't have on the gap list:
- **APM service map** (auto from `ddtrace`) — see mediator → relay → backend visually
- **Continuous Profiler** (replaces py-spy/pprof) — live CPU/heap profiles per customer
- **DD Watchdog** — auto anomaly detection on metrics; no manual thresholds
- **Database Monitoring** — slow query log, locks, table size growth (would have caught the `archive_by_server_id` issue)
- **Live Processes** — see gunicorn workers, RQ subprocess across all customer clusters
- **Network Performance Monitoring (eBPF)** — TCP retransmits, conn failures (would have caught the sentinel-port-not-open issue in 30s)
- **Trace-log correlation** — one trace ID through mediator → relay → backend; click trace, see all logs

## 7. Multi-tenancy decision

Load-bearing decision before any code:

| Option | Setup | Trade-off |
|---|---|---|
| **A. Vendor's single DD org** | One DD API key in chart (encrypted), tag everything with `customer.id` | Simplest. We pay DD bill. Customers must accept telemetry leaving cluster — but routed through relay, not direct |
| **B. Customer's own DD org** | Customer provides their API key | Customer pays. We lose centralized visibility |
| **C. Hybrid** — relay-side OTel collector with two exporters | Two DD destinations | Most flexible. Customer security team must approve |
| **D. Vendor org but per-customer scrubbing in relay-side OTel collector** ⭐ | Strip business-payload fields before export, keep operational metrics+logs | Best balance for "see all customers' health" without leaking command bodies |

## 8. File-level changes (when we proceed)

### Mediator (`/Users/axe/blitzy/archie-client-mediator`)

| Change | File | Effort |
|---|---|---|
| Wrap gunicorn with `ddtrace-run` | `Dockerfile:76` | 5 min |
| Add `datadog` (statsd) for custom metrics | new `src/utils/metrics.py` | 1 hour |
| Emit metrics from drain loop | `src/services/ws_client.py:513-545` | 30 min |
| RQ worker heartbeat metric | `main.py:184-202` | 30 min |
| Watchdog cause-of-death log + metric | `main.py:213-216` | 15 min |
| Deep `/healthz` endpoint | `src/api/routes/utils.py` (new route) | 1 hour |
| `requirements.txt` — add `ddtrace`, `datadog` | one-liner | 1 min |
| Stop swallowing OTLP exceptions | `src/api/routes/otlp.py:102` | 15 min |
| Downgrade disconnect ERRORs to WARNINGs | `src/services/ws_client.py` (search "Disconnected from relay") | 15 min |
| Preserve `client_id` through disconnect | `src/services/ws_client.py` — don't clear on disconnect, only on re-register | 1 hour |
| `/otlp` endpoint accept metrics + traces (currently logs only) | `src/api/routes/otlp.py` | 2 hours |

### Relay (`/Users/axe/blitzy/archie-service-relay`)

| Change | File | Effort |
|---|---|---|
| Wrap gunicorn with `ddtrace-run` | `Dockerfile` (whichever line runs gunicorn) | 5 min |
| Emit connection-count gauge from registry | `registry_service.py:110, 127` | 30 min |
| Emit drain pool gauges | `drain_pool.py:151, 164-180` | 30 min |
| Emit per-client queue depth | `request_queue_service.py:53-54` | 30 min |
| Stale-mediator detection timer | new file or `registry_service.py` | 2 hours |
| Chunk-buffer leak metric | `control.py:_chunk_buffers` | 30 min |
| New `/metrics`, `/traces` namespaces (or `/telemetry`) | `src/namespaces/` | 1-2 days |
| Forward OTLP to a relay-side OTel collector (Option 3) | new namespace handler + deploy collector alongside relay | 2-3 days |

### Helm chart (`/Users/axe/blitzy/archie-helm-chart/blitzy-client/chart`)

| Change | File | Effort |
|---|---|---|
| Add traces pipeline to OTel collector | `values.yaml:264-285` (existing `service.pipelines:` block) | 30 min |
| Inject DD-related env vars (if Option 1/2/4) into mediator | `templates/client-mediator.yaml:93-145` — extend `extraEnv:` from CSI-synced Secret | 30 min |
| Add `app.kubernetes.io/version` label | `Chart.yaml` add `appVersion: ...`; helper at `_helpers.tpl:84-91` already templates it | 5 min |
| Add liveness/readiness probes to mediator | `templates/client-mediator.yaml` | 10 min |
| Open metrics port on mediator service (only if scraping path used) | `templates/client-mediator.yaml` Service block | 5 min |
| `extraManifests:` template that takes the array from values.yaml line 1 and renders each element verbatim | new `templates/extra-manifests.yaml` | 30 min |

## 9. Open questions to settle before code

| # | Question | Why it matters |
|---|---|---|
| 1 | Logs only, or logs + metrics + traces? | Traces are 10× volume; metrics need cardinality control |
| 2 | Single shared DD org or per-customer? | Affects tagging, billing, isolation |
| 3 | Where does PII scrubbing happen — mediator or relay? | Mediator-side is "stop bad data leaving customer". Relay-side is "stop bad data being indexed". Customers may insist on the former |
| 4 | Sampling strategy — head-sample, tail-sample on errors, all? | Affects cost and debuggability |
| 5 | Trace propagation across the WS boundary | `ddtrace` injects `x-datadog-trace-id` on mediator side; relay must propagate to backend. Today: doesn't |
| 6 | Volume estimate at scale | Existing `/logs` works because logs are batched; metrics/traces need similar batching discipline |
| 7 | What if telemetry channel itself is broken | Today: outbound queue. But if queue is full of telemetry, application traffic gets blocked. Need separate quota |
| 8 | Auth for telemetry channel | Customer's mediator auths to relay. Need additional gating to prevent misbehaving customer flooding our DD bill |
| 9 | Retention vs customer access | DD retention vs customer's own log retention. Compliance: does telemetry round-trip back to customer? |
| 10 | Air-gapped customers | Even relay traffic blocked. Fallback: local-only Loki/Tempo/Mimir/Sentry-on-prem |
| 11 | Does existing `OTEL_FORWARDER_URL` Cloud Run service already deliver to DD, or just to current log backend? | If already DD-bound for logs, we're 80% there for free |
| 12 | Where does vendor-side dashboard live — single DD console or per-customer? | Affects routing config in forwarder/relay |
| 13 | Is `OTEL_FORWARDER_URL` separate from the relay process or same? | If separate, upgrading to a full OTel collector there is the natural Option 3 |

## 10. Cross-cutting issues we found mid-discussion (not strictly observability, but blocking)

These came up while debugging and need to be on the radar:

| Issue | Where | Status | Doc reference |
|---|---|---|---|
| **Dual `client_id` for one mediator** — `key_last_4` lookup with no `ORDER BY`, no UNIQUE on `key_hash`, returns multiple rows non-deterministically | admin service `api_key_repository.py:52-66` | **Open — root cause identified** | This doc §10; verification SQL provided in chat |
| **Vault HA TLS deadlock under Argo** | chart's `post-install.yaml` is PostSync; Vault is Sync — phase cycle | Workaround (manual cert bootstrap) | `INSTALL_TROUBLESHOOTING.md §2` |
| **CNPG operator RBAC** — ClusterRoleBinding subject namespace bug + missing namespace Role | chart RBAC | Workarounds applied | `INSTALL_TROUBLESHOOTING.md §1` |
| **Helm `enabled: 'false'` (string) doesn't disable** | chart values everywhere | Documented gotcha | `INSTALL_TROUBLESHOOTING.md §8` |
| **Bitnami sentinel image `8.6.0` not on Docker Hub** | Bitnami publishing change | Use `redis:7.2` for both | `INSTALL_TROUBLESHOOTING.md §5` |
| **Helm `charts/` auto-discovery** — subcharts render even without `Chart.yaml` deps | chart structure | Documented gotcha | `INSTALL_TROUBLESHOOTING.md §8` |
| **Argo sync-wave only orders within phase** | chart hooks | Documented gotcha | `INSTALL_TROUBLESHOOTING.md §8` |

## 11. What we might still be missing

After reviewing the discussion, these are areas we haven't touched yet that could be load-bearing:

| Area | Why it matters | Suggested action |
|---|---|---|
| **PII scrubbing pipeline** — concrete OTel processor config to redact bash command stdout/stderr, secrets, PR titles, repo paths | Customer security teams will block install if command bodies appear in vendor's DD | Define explicit list of fields to drop/hash before export |
| **Telemetry rate limiting per customer** — soft and hard caps to protect vendor DD bill from a single misbehaving customer | One customer's 100× normal volume can cost $$$ | Cap at relay-side OTel collector with `memorylimiter` + tail-sampling rules |
| **Trace context propagation across Socket.IO `/control` requests** | Today, mediator → relay → backend has zero trace correlation. Without this, APM service map is broken | Add `traceparent` / `tracestate` to `request_response` payload schema |
| **Telemetry queue isolation from application queue** | If telemetry buffer fills up, command traffic shouldn't be blocked. Today: shared `mediator:outbound_queue` | Separate Redis list/stream for telemetry, with separate drain quota |
| **Backpressure signal to OTel collector** | When mediator can't forward to relay, OTel collector keeps batching and OOMs the customer's node | OTel collector `memory_limiter` processor + clear failure metric |
| **`extraManifests:` template implementation** | The values.yaml line 1 has the array but no chart template renders it yet (assumed) | Add `templates/extra-manifests.yaml` that loops and emits each entry |
| **Helm `appVersion` populating `app.kubernetes.io/version` label** | Without this, k8s tooling can't tell apart blitzy-client 1.0.34 from 1.0.36 in metrics aggregation | Set in `Chart.yaml` per release |
| **Per-customer log retention** — vendor's DD vs customer's compliance requirements | Some customers must retain their own logs for N years; DD vendor org has different retention | Decide: round-trip back to customer, or customer keeps separate copy via OTel fan-out |
| **Sensitive label filtering at OTel collector** — pod labels currently auto-extracted via preset `extractAllPodLabels: true` | Customer pod labels can contain anything (including secrets in misconfigured envs) | Allowlist of labels to forward |
| **Dashboards as code** — DD dashboards/monitors versioned alongside the chart | Customer-onboarding teams need a one-click "create dashboards for new customer" | Terraform DD provider + per-customer template |
| **Alert routing per customer** — vendor's PagerDuty rotation needs to know which customer's alert is firing | Without this, every alert says "blitzy-client unhealthy" with no context | Tag alerts with `customer.id`, route to per-customer Slack channels |
| **Cold-start install diagnostics** — first-time install failures need their own pipeline because the mediator may never start successfully | If mediator never connects, no telemetry flows. Need fallback (helm pre-install Job that posts a heartbeat to a vendor endpoint with install context) | Helm pre-install diagnostic Job |
| **Worker pod telemetry** — workers spin up, run for minutes, get reaped. Their logs are gone | OTel filelog catches them while running, but post-mortem is hard | Worker emits structured "job lifecycle" metrics that survive pod death |
| **Runner pod telemetry** — same issue, plus runners are even more ephemeral | Same fix | Same fix |
| **CDP tunnel telemetry** — Chrome browser hangs in customer pod are notoriously hard to diagnose | First report is "the screenshot tool is stuck" with no further info | Heartbeat already exists (CDPProxy `_heartbeat_loop`); emit it as a metric |
| **Migration job telemetry** — currently fires once at install; no visibility into success | If migration silently fails, mediator starts but database is wrong | Migration job emits structured "migration applied: X" log |

## 12. Recommended minimum viable rollout (when we proceed)

In order, because each step builds on the previous:

**Phase 0 — Quick wins (1 day, no architectural change)**
- Watchdog cause-of-death log
- Downgrade disconnect ERRORs to WARNINGs
- Preserve `client_id` through disconnect
- Add deep `/healthz` to mediator
- Add liveness/readiness probes to mediator deployment
- `app.kubernetes.io/version` label populated

**Phase 1 — Stand up the channel (3-5 days)**
- Decide Options A/B/C/D for multi-tenancy
- Stand up relay-side OTel collector (Option 3)
- Add traces pipeline to customer-side OTel collector
- Wire `ddtrace-run` on both mediator and relay
- Trace propagation across WS boundary

**Phase 2 — Custom metrics (2-3 days)**
- DogStatsD wrapper in mediator + relay
- Emit drain loop, queue depth, RQ worker, connection count, watchdog kill, drain pool stats
- Build first dashboard (per-customer health summary)

**Phase 3 — Alerting + synthetics (2 days)**
- DD monitors for: mediator down, queue backing up, error rate spike, connection flapping
- DD Synthetics polling relay's `/health/connections` and `/health/queue`
- App-internal canary

**Phase 4 — Long tail (open-ended)**
- Sentry-equivalent exception capture
- Support-bundle endpoint
- Feature flags
- Remote diagnostic command channel
- Migration visibility
- Per-customer dashboards as code
- Air-gapped customer support (Loki/Tempo/etc)

---

## Decisions still owed before any code

1. **Do we own a single DD org or one per customer?** (Question 2 above)
2. **Where does PII scrubbing happen — mediator or relay?** (Question 3)
3. **Is `OTEL_FORWARDER_URL` already DD-bound for logs?** If yes, we're already 80% there. (Question 11)
4. **Are we OK with telemetry consuming Socket.IO bandwidth on the same channel as application traffic, or do we want a separate channel?** (Question 7)
5. **Air-gapped customer story — is there an air-gapped customer in the pipeline, or only internet-connected for now?** (Question 10)

Settle 1, 3, and 4 in a single short meeting and we can start Phase 0 the same day.
