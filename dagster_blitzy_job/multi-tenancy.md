# Multi-tenancy — what each orchestrator isolates

> Companion to `orchestrator-comparison.md`. Captures the multi-tenancy
> discussion (2026-05-01) and the Blitzy-specific recommendations. The
> headline goes into the deck (`slidev/slides.md` — slide
> "Multi-tenancy — what each engine isolates").

## The three flavors

Pick the model first; the engine question is downstream.

| Model | Who's isolated | What's shared | Typical use |
|---|---|---|---|
| **Soft** | UI views, run history, resource attribution | One control plane, one metadata DB, one cluster | Internal-only orchestrator, trusted tenants |
| **Hybrid** | Compute (executors, namespaces, clusters) + data | Control plane, scheduler, UI | Most production multi-tenant orchestrators |
| **Hard** | Everything — separate deployment per tenant | Nothing | Regulated verticals, large customer separation |

For Blitzy today: the orchestrator UI is **internal**, customers never log in, and per-company Neo4j / GCS / GitHub creds already isolate the data layer. That's hybrid multi-tenancy — and the cheapest configuration of all three.

---

## Dagster — multi-tenancy primitives

| Primitive | What it does | Tenancy use |
|---|---|---|
| **Code locations** | Each is a separate gRPC server with its own `Definitions`, deps, Python version | Per-tenant or per-tier code isolation. A bug in one location can't crash another. |
| **Resources scoped to a `Definitions`** | Typed Python objects holding Pub/Sub topics, GCS buckets, Neo4j creds, etc. | Different topic / bucket / Neo4j URI per tenant — composed at workspace load time, not at op-call time |
| **Run tags** (`customer_id`, `org`, `tier`) | Free-form key/value attached to a run | UI filters, GraphQL queries, dashboards, and `tag_concurrency_limits` |
| **Workspaces** | One per deployment | If you want hard isolation between tenants → multiple deployments |
| **Dagster+ Teams + RBAC** | Role-based access on code locations, assets, and runs | The only way to scope UI visibility per tenant. **Not in OSS Dagster.** |
| **Branch Deployments** (Dagster+) | Ephemeral per-PR Dagster instances | Dev/preview, not tenant isolation |
| **K8s launcher config** (`K8sRunLauncher`, `K8sJobLauncher`) | Per-resource-bundle namespace, ServiceAccount, NetworkPolicy | Hard runtime isolation — per-tenant or per-tier namespace |

### Where Dagster is strong

- **Code locations + per-Definitions resources** give clean code-and-data isolation without spinning up a separate control plane per tenant.
- **Tag-based concurrency** means one rule (`{customer_id} → 3`) covers throttling for all tenants — no Pool matrix to provision.
- **Typed Pydantic resources** are an ergonomic win for per-tenant config. `Neo4jResource(uri=..., username=..., password=...)` per Definitions beats stringly-typed Connections.

### Where Dagster is weak

- **OSS has no built-in user model.** Dagster+ Teams is the only RBAC option, and it's a paid feature.
- **Single-pane UI by design** — you can filter by tag, but you can't *prevent* a viewer from seeing other tenants' runs without Dagster+.
- **Metadata DB is shared.** Anything you put in run tags or `MetadataValue` is visible to anyone with UI access.

---

## Airflow — multi-tenancy primitives

| Primitive | What it does | Tenancy use |
|---|---|---|
| **DAG-level RBAC** (`access_control` on `DAG`) | Per-DAG read/edit permissions per role | Real enforcement — a feature OSS Dagster doesn't have |
| **Roles + custom permissions** (FAB-based) | Mint custom roles, grant per-DAG | Map "Tenant Acme Viewer" → only Acme DAGs |
| **Pools** | DB-backed slot allocations | Per-tenant resource caps |
| **Variables / Connections** | Key/value store in the metadata DB | Convention-based prefixing (`acme_neo4j_uri`). **No native scoping.** |
| **Tags on DAGs** | UI filter | Cosmetic, not enforcement |
| **Multi-scheduler HA** | N schedulers share Postgres lock | Scaling, not tenancy — all schedulers see all tenants |
| **Edge Executor (3.0)** | Tasks run on tenant-side agents, off the central control plane | Strongest hybrid-tenancy lever Airflow has shipped recently |
| **Astronomer Software / Composer / MWAA** | Vendor-managed multi-team workspaces | Formal multi-tenancy on managed offerings — not OSS |

### Where Airflow is strong

- **DAG-level RBAC out of the box** — has been there since 1.10, no paid tier required.
- **Pools are well-understood** for per-tenant resource caps.
- **Edge Executor (3.0)** moves task execution to tenant-controlled agents — the cleanest path to hybrid multi-tenancy in OSS orchestration today.

### Where Airflow is weak

- **Connections and Variables are global namespaces.** Sensitive data leaks if you don't enforce naming hygiene. No per-tenant scoping at the metadata layer.
- **Scheduler sees and parses all tenants' DAGs together.** No tenant-level isolation at the scheduler / metadata DB layer in OSS Airflow.
- **Multi-tenancy is convention-heavy.** Working setups exist, but they're glued together by naming conventions plus RBAC plus DAG-folder discipline — not one coherent primitive.

---

## Side by side

| Capability | Dagster | Airflow 3 |
|---|---|---|
| Per-tenant code isolation | Code locations (gRPC servers) | DAG folders + `access_control` |
| Per-tenant resource bundles | Typed `Definitions`-scoped resources | Connections + Variables (global namespace, prefix by convention) |
| Per-tenant compute isolation | K8s namespace via launcher resource | KubernetesExecutor namespace + Edge Executor (3.0) |
| Native RBAC in OSS | ❌ — Dagster+ only | ✅ — DAG-level since 1.10 |
| Run-history visibility scoping | Tag filter (cosmetic) or Dagster+ Teams | DAG `access_control` (real enforcement) |
| Per-tenant secret isolation | Resource config injects per-Definitions | Convention only; same Variables table |
| Tenant-level throttling | One `tag_concurrency_limits` rule | One Pool per tenant |
| Metadata-leak surface | Tags, `MetadataValue`, asset metadata | Variables, Connections, XCom |
| Hard isolation cost | Multiple deployments | Multiple deployments |

Both engines hit "soft + hybrid" out of the box. The difference is *where* the abstractions are typed (Dagster) vs. convention-based (Airflow).

---

## Applied to Blitzy

The current state already implements hybrid multi-tenancy at the data layer:

| Resource | Per-customer? | How |
|---|---|---|
| Neo4j instance | ✅ | `get_company_neo4j_instance_credentials(company_id)` at runtime |
| GCS prefix | ✅ | `AdminStorageService(project_id, task_id, ...)` keyed off project |
| GitHub creds | ✅ | `GITHUB_SECRET_SERVER` resolves per company |
| Pub/Sub topics | ⚠️ Per-environment, not per-customer | One set of topics per env (dev/qa/stage) |
| LangSmith project | ⚠️ Per-environment | `LANGSMITH_PROJECT=blitzy-dev` |
| K8s namespace | ❌ | Single `default` namespace today |

The orchestrator's job is therefore:

1. **Route the right resources to the right run.** Keyed off `EVENT_DATA.company_id`.
   - Dagster: resource factory reads run-config, builds the right `Neo4jResource`, `StorageResource`.
   - Airflow: `BranchPythonOperator` + Connection name lookup (`Connection.get(f"{company_id}_neo4j")`).
2. **Throttle per-tenant** so one customer can't hog all build slots.
   - Dagster: `tag_concurrency_limits = {"customer_id": 3}`
   - Airflow: Pool per customer
3. **Hide tenant A's runs from tenant B's users** — only matters once external users are in the UI.
   - Dagster: Dagster+ Teams (paid)
   - Airflow: DAG `access_control` (free)
4. **Limit per-tenant cluster blast-radius** so a Pod stampede in one tenant doesn't starve another.
   - K8s namespace per **tier** (not per customer — see below).

---

## Recommendations

1. **Stay soft multi-tenant in the control plane.** One deployment per env (dev / qa / stage). Tenants distinguished by `company_id` run tags. Build for external-customer access only when it actually shows up on the roadmap.
2. **Hard-isolate at the resource layer** — already correct. Don't merge per-customer Neo4j / GCS / GitHub.
3. **Hard-isolate at the k8s namespace layer per tier, not per customer.** Per-customer namespaces don't scale: 50+ namespaces × `ResourceQuota`s × `NetworkPolicy`s × ServiceAccounts × imagePullSecrets = ops nightmare. Per-tier (`blitzy-tier-enterprise`, `blitzy-tier-standard`) gives you the noisy-neighbor isolation without the sprawl.
4. **Don't put PII in tags or `MetadataValue`.** Treat the metadata DB as a soft-public surface. Anything sensitive lives in customer-owned storage and is referenced by ID.
5. **Use typed run tags as the tenant key.** Every run gets `{"customer_id": ..., "org_id": ..., "tier": ...}` set by the sensor or trigger. These drive throttling, dashboards, and cost attribution in one shot.
6. **For per-tenant LangSmith / observability**, switch to per-customer LangSmith projects only if customers ever need access to traces. Today (internal only) one project per env is fine.

## Trigger conditions to revisit

| If this happens | Move to | Why |
|---|---|---|
| External customers log into the orchestrator UI | Dagster+ Teams **or** Airflow with DAG `access_control` | Tag-based filtering is not access control |
| Onboard a regulated-vertical tenant (HIPAA, FedRAMP, ITAR) | Dedicated deployment for that tenant | Shared metadata DB likely fails the audit |
| One tenant's burst regularly starves others, even with tag concurrency | Per-tier k8s namespace + per-tier executor | The control plane isn't the bottleneck — the cluster is |
| Run count > 5k concurrent across tenants | Multi-daemon sharding (Dagster) or multi-scheduler HA (Airflow) | One daemon / scheduler can't keep up |
| Per-tenant SLA reporting becomes a contract requirement | Per-tenant LangSmith project + Dagster GraphQL filter on `customer_id` | Tag filtering at the dashboard layer is enough; full per-tenant deployment is overkill |

Document these triggers in the runbook so the upgrade decision isn't made in panic during an incident.

---

## Where each engine pushes back

- **Dagster pushes back if** you need OSS RBAC. There isn't any. You buy Dagster+ or you build a reverse proxy with auth and accept that anyone past it sees everything.
- **Airflow pushes back if** you need typed per-tenant resources. You'll end up with `acme_neo4j_uri`, `acme_neo4j_password`, `acme_gcs_bucket` Variables/Connections sprawled across the metadata DB. Workable, but typed Pydantic resources in Dagster are a real ergonomic win for this exact use case.

## Headline for the deck

**For Blitzy's pattern (per-company external resources, internal-only orchestrator UI), Dagster's typed-resource-per-Definitions model is the natural fit, and OSS RBAC isn't needed yet. Customer-facing UI on the roadmap = trigger to evaluate Dagster+ Teams.**

---

## See also

- `orchestrator-comparison.md` — full Dagster vs Airflow 3 vs Argo evaluation
- `phase1-implementation.md` — Phase 1 spec including per-Definitions resource bundles
- `env-contract.md` — per-tenant env vars currently injected via `EVENT_DATA`
- `slidev/slides.md` — deck slide "Multi-tenancy — what each engine isolates"
