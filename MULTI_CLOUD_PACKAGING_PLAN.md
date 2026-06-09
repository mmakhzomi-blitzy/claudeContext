# Blitzy Multi-Cloud Packaging — Plan & Categorization

**Prepared:** 2026-05-20
**Author:** mohammed@blitzy.com
**Scope:** Package Blitzy services for multi-cloud deployment (AWS / GCP / Azure) via a single Helm chart, with pluggable interfaces so services run against either bundled components or customer BYO infrastructure.

---

## 0. Guiding principles & constraints

- **Deployment surface:** Kubernetes + Helm only. No Argo, ECS, Cloud Run, Harness, or Cloud Run Jobs.
- **Two install modes per component:** *bundled* (we ship it in the chart) and *BYO* (customer-managed). Every infra dependency must support both.
- **Cloud agnosticism is achieved through interfaces**, not branches. Code talks to `PubSub`, `Secrets`, `ObjectStorage`, `Queue` abstractions; cloud-specific implementations sit behind them.
- **Blitzy client (mediator) should keep working unchanged** if migration is done correctly — its contract is the validation gate.
- **One driver of the design** is enabling our own GCP→AWS migration via this Helm chart (DNS cutover only).
- **Two product surfaces** to keep distinct: *Blitzy Shared Clients* vs *Blitzy BlackBox*. Decide per-feature which surface it ships to.

---

## 1. Categorization

The brain-dump items group into **9 workstreams**. Streams marked **P** can run in parallel from day one; streams marked **S** are sequenced behind a prerequisite.

| # | Workstream | Mode | Prerequisite |
|---|---|---|---|
| 1 | **Helm Chart Foundation** | P | — (start now, unblocks others) |
| 2 | **Code Refactor: Cloud Interfaces** | P | — (can start now, independent of Helm) |
| 3 | **Service Migration to K8s** | S | 1 (skeleton) + 2 (interfaces for that service) |
| 4 | **Platform Jobs Migration** | S | 1 + 2 + queue decision |
| 5 | **Observability** | P→S | Datadog prep is P; per-service wiring is S behind 3 |
| 6 | **Licensing** | P | — |
| 7 | **Phone-Home Telemetry** | S | 3 (services emit) + decision on what we collect |
| 8 | **Specialty Helm Charts** | P | — (neo4j, langsmith separate) |
| 9 | **Cross-cutting: Dev envs, Docs, Sizing, Obfuscation, E2E tests** | mixed | varies |

---

## 2. Workstream detail

### Workstream 1 — Helm Chart Foundation  *(start immediately, parallel to everything)*

The skeleton needs to land first so service migration (WS3) and observability wiring (WS5) have a target to deploy into.

- **Skeleton**
  - Chart layout for all Blitzy services (templates, values schema, sub-charts).
  - Per-service service accounts (no shared SA).
  - Windows Job support (template + node selector pattern for Windows pool).
  - Datadog values stubs (real wiring in WS5).
- **K8s service discovery** — update `blitzy-util` service client to resolve via K8s DNS/Services instead of hardcoded URLs. *Depends on skeleton existing first.*
- **Integrate the archie services** into the chart (templated Deployment/Job manifests, not live migration yet — WS3 owns the actual cutover).
- **Bundled component sub-charts** — vendored or dependency-pinned:
  - Postgres
  - Redis (session storage only; **not** RQ)
  - ELK
  - Vault
  - Red Panda
- **BYO component support** — values surface + interface routing for:
  - RDS
  - Customer-managed Redis
  - GSM / Vault / AWS Secrets Manager
  - S3 / GCS / MinIO
  - Customer-managed Red Panda (or Kafka — see Open Q)
- **Helm chart documentation** — install, upgrade, values reference, BYO matrix.

### Workstream 2 — Code Refactor: Cloud Interfaces  *(start immediately, parallel to WS1)*

Independent of Helm — pure code changes that make services portable.

- **Define interfaces:**
  - Pub/Sub
  - Secrets (lives in `blitzy-util` vault util)
  - Object Storage
  - Queue (Red Panda / Kafka)
- **Implementations behind each interface:**
  - Pub/Sub → Kafka *and* Red Panda (future: SQS, GCP Pub/Sub)
  - Secrets → Vault, GSM, AWS Secrets Manager
  - Object Storage → S3, GCS (MinIO via S3 API)
- **Remove dependency on GCP Cloud Functions.**
- **`archie-service-light`** refactor.
- **Decide:** does `archie-platform-event-listener` need a refactor before it can sit behind the new pub/sub interface, or can it adopt it as-is? *(flagged in source list with a "refactor ?")*

### Workstream 3 — Service Migration to K8s  *(sequenced behind WS1 skeleton + WS2 interface for that service)*

Migrate each service to run as a K8s Deployment under the Helm chart. Order roughly by blast radius (lowest first):

- `archie-service-markdown`
- `archie-service-admin`
- `archie-github-handler`
- `archie-job-tracker-ui`
- `archie-job-tracker`
- `archie-service-backend`
- `archie-platform-event-listener` *(may need WS2 refactor first)*
- **Relay Service** — must be configurable (per the source list)
- **DB migrations** — package as a pre-install/pre-upgrade Helm hook

### Workstream 4 — Platform Jobs Migration  *(sequenced behind WS1 + WS2 + queue decision)*

The seven `archie-job-*` services + chat + pdf. These currently run on Cloud Run Jobs and chain via Pub/Sub — see [PLAN.md](PLAN.md) for the existing Dagster-on-jobs plan; align with whatever direction is chosen there.

- code-downloader
- code-graph-generator
- reverse-document-generator
- reverse-file-mapper
- reverse-thinking-generator
- reverse-code-generator
- generate-pdf
- Chat

### Workstream 5 — Observability

- **Datadog integration** (parallel prep with WS1)
  - Wire agent + values stubs in the Helm chart.
  - Per-component testing & verification plan.
  - **Replace Captain Panic with Datadog** across services.
- **ELK alternative path** (decision still open)
  - ELK as side service + logging path *vs.* Datadog-only.
  - Used to also replace Captain Panic where Datadog is not the choice (customer-hosted scenarios).

### Workstream 6 — Licensing

- **Blitzy Licensing service** — JWT API key issuance & validation.
- Decide product surface (Shared Clients vs BlackBox) and whether licensing is enforced inside services or via a sidecar/gateway.

### Workstream 7 — Phone-Home Telemetry  *(sequenced behind WS3)*

Open scope — needs decisions on what we collect back:

- Metering (usage units, billing inputs)
- Job info (for job tracking visibility to us)
- Logs? *(open question — depends on customer-hosted constraints)*

### Workstream 8 — Specialty Helm Charts  *(separate from main chart)*

- **neo4j** — separate Helm chart.
- **langsmith** — separate Helm chart.

### Workstream 9 — Cross-cutting

- **Dev environments:** AWS, GCP, Azure (one per cloud; needed for WS3/WS4 validation).
- **Sizing:** per-service resource requests/limits, node pool recommendations.
- **Code / container obfuscation:** for BlackBox distribution.
- **Documentation:**
  - Architectural document
  - Cost estimates (customer-facing)
  - Cost of Datadog vs BYO observability
- **End-to-end automated tests:** per service — MM to enumerate the set.

---

## 3. Dependency map (compact)

```
WS1 Helm skeleton ──┬──> K8s service discovery (blitzy-util)
                    ├──> WS3 Service migration ──> WS7 Phone-home
                    ├──> WS4 Jobs migration
                    └──> WS5 Datadog wiring

WS2 Interfaces  ────┴──> WS3, WS4 (each service adopts interfaces during its migration)

WS6 Licensing       (independent)
WS8 neo4j/langsmith (independent)
WS9 Dev envs        (must precede WS3/WS4 validation)
WS9 E2E tests       (per service, follows that service's WS3 migration)
```

---

## 4. Open questions & decisions needed

These came directly from the planning notes — flagged here so they don't get lost.

1. **Queue: Kafka vs Red Panda** for the bundled default. Future support for SQS and GCP Pub/Sub is in scope but not blocking.
2. **Logging for customer-hosted deployments** — what's the supported path when Datadog isn't an option?
3. **What do we phone home?** Metering / jobs / logs — needs an explicit list.
4. **GCP→AWS migration via this Helm chart** — confirm DNS cutover is the only switch needed. *(Assumption: yes if migration is done properly.)*
5. **Blitzy client (mediator) compatibility** — validate it works unchanged against the new packaging.
6. **Helm chart BYO matrix** — confirm every component has a working BYO path *and* a working bundled path.
7. **Development model** — which features ship to Blitzy Shared Clients vs Blitzy BlackBox? Needs a per-feature call.
8. **`archie-platform-event-listener`** — refactor required before pub/sub interface adoption, or not?
9. **Relay service configurability** — what knobs need to be values-driven?

---

## 5. Suggested kickoff (next ~2 weeks)

Three things start in parallel:

1. **WS1**: Helm skeleton + chart layout + service account model.
2. **WS2**: Interface definitions (pub/sub, secrets, object storage, queue) — code only, no implementations yet.
3. **Decision sprint**: resolve Open Questions 1, 2, 3, 7 — they gate WS4 and WS7 scope.

Once WS1 skeleton lands → unblock K8s service discovery util update → start WS3 with `archie-service-markdown` as the lowest-risk pilot.
