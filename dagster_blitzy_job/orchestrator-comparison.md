# Orchestrator comparison — Dagster, Airflow 3, Argo Workflows

> Companion to `phase-plan.md`. Consolidates the three-way evaluation
> used to justify the Dagster pick, with the Airflow 3.0 updates and
> the Argo deep-dive.

## TL;DR for Blitzy

| # | Tool | Verdict |
|---|---|---|
| 1 | **Dagster** | Best fit — asset model, typed metadata, Python-first, Branch Deployments |
| 2 | **Airflow 3** | Closed much of the gap; only wins if we need non-Python tasks |
| 3 | **Argo Workflows** | Different category; we'd rebuild the metadata/UI layer — net negative |

The prior ranking (Dagster > Airflow > Argo) holds. The margin between
Dagster and Airflow 3 is **narrower** than pre-3.0 and should be
revisited in 12 months.

---

## 1. Airflow 3.0 — what changed (April 2025)

Airflow 3.0 is the biggest release in the project's history. For this
evaluation, five features matter:

### 1.1 Asset-based scheduling

Airflow 3 adds **assets** as first-class scheduling units. Tasks can
produce assets; downstream tasks trigger when upstream assets are
updated. Closes roughly 60% of the Dagster asset advantage — but the
lineage UI is still maturing and typed I/O / I/O managers remain
Dagster-only.

### 1.2 Task SDK + Edge Executor — multi-language support

Airflow 3 decouples the task runtime from the scheduler via a **Task
SDK**. Implications:

- Tasks can run in isolated, remote, or containerised environments
  — no dependency conflicts with the scheduler
- **Go** is the first non-Python language with a production SDK
- **Java** and **R** are on the roadmap
- Edge Executor runs tasks on a remote agent with minimal setup

This is the biggest Airflow 3 win and the one that *could* flip the
ranking if Blitzy ever onboards non-Python jobs (ML infra in Go, data
quality scripts in R, JVM services). For today's seven-job Python
fleet, it doesn't apply.

### 1.3 React UI rewrite

The Flask webserver is replaced by a modern React UI. Run + task
drilldown is noticeably better; feature parity with Dagster's UI on
navigation, still behind on metadata rendering.

### 1.4 DAG versioning

Git-like history for DAG definitions — a long-standing ask.
Dagster's equivalent is code locations with hot reload plus external
git history.

### 1.5 Stable DAG authoring API

Breaking changes now go through deprecation cycles. This reduces the
upgrade tax that made Airflow painful historically.

### What Airflow 3 still does **not** have (vs Dagster)

- Zero-infra local dev (`dagster dev` still unique — Airflow still
  needs the scheduler + DB + webserver)
- Typed I/O as a first-class concept (Xcom remains the main
  data-passing primitive)
- Branch Deployments (ephemeral per-PR envs)
- Asset lineage as a UI-native concept (3.0 has assets but the
  lineage view is evolving)
- Asset freshness policies with auto-materialize
- I/O managers (pluggable storage per op / asset)

---

## 2. Argo Workflows — the deep-dive

> **Plan clarification (2026-05-21):** the Multi-Cloud Packaging Plan §0 line 11
> ("No Argo, ECS, Cloud Run, Harness, or Cloud Run Jobs") refers to **Argo CD**
> (the GitOps deployment tool), **not Argo Workflows**. Argo Workflows is
> therefore on the table as a workflow-orchestration candidate, especially when
> BlackBox/customer-hosted portability is in scope — see §2.4 trade-offs and
> §3.3 ranking caveat below.

### 2.1 What it actually is

- Kubernetes-native workflow engine
- Every step is a container Pod
- Workflows defined as YAML CRDs (`Workflow`, `WorkflowTemplate`,
  `ClusterWorkflowTemplate`, `CronWorkflow`)
- Sequential, DAG, or parametrised execution
- CNCF graduated (2022); adopted at BlackRock, Intuit, Red Hat
- Sister projects: Argo CD (GitOps), Argo Events (event-driven
  triggers), Argo Rollouts (progressive delivery)

### 2.2 Where Argo genuinely wins

- **Pure k8s-native.** Just a controller Pod; no control plane to
  operate beyond it. The lightest footprint of the three.
- **Language-agnostic.** Any container is a step. If your workload is
  truly polyglot, this is a big deal.
- **Massive parallelism.** Proven at thousands of concurrent
  workflows. Fan-out to thousands of Pods is the default case, not a
  stress test.
- **GitOps-native.** YAML + Argo CD is a natural combo. Every
  workflow is a reviewable artifact in Git.
- **Mature ecosystem** on k8s — deep integration with PVCs, service
  accounts, image pull secrets, sidecars.

### 2.3 Where Argo falls short for Blitzy

- **No assets, no lineage, no data-awareness.** The concepts aren't
  in the model. The BillingReport / %LOC / tokens story would live
  *outside* the engine.
- **No typed I/O.** Outputs are files in an artifact repo
  (S3 / GCS / etc). Passing a structured Python object from one step
  to another means writing JSON and reading it back.
- **YAML-heavy.** Seven chained workflows are maintainable, but each
  refactor is a textual diff across YAML files with no type checking.
- **Minimal UI metadata.** The Argo UI shows the DAG and a Gantt, plus
  Pod logs. No equivalent to Dagster's `MetadataValue` for
  `tokens_input`, `hours_saved`, `percent_LOC`.
- **No Python-native testing.** Argo workflows test against a real
  cluster (or `kind` / `minikube`). You can't unit-test a step the
  way Dagster lets you test an `@op`.
- **No hooks primitive.** The AI-triage pattern from
  `ai-integration-options.md` would need a **sidecar microservice**
  subscribing to Argo Events — that's a full extra service to run.
- **Smaller active contributor base.** The 2025 orchestration
  ecosystem survey places Argo in a "warning zone" vs Dagster and
  Airflow on contributor activity. Not abandonment, but slower
  roadmap.

### 2.4 Trade-off table

| Concern | If we picked Argo | If we stay with Dagster |
|---|---|---|
| Ops footprint | Lightest (controller only) | Control plane + Postgres |
| Wrapping existing jobs | Zero wrapper code — containers are steps | `k8s_job_op` wrapper (still small) |
| Billing metadata | External system; rebuild UI on artifact files | First-class `MetadataValue`; UI renders it |
| Asset lineage | External docs / manual | Native asset graph |
| Multi-env | Standard k8s namespaces | Dagster `Definitions` per env + Dagster+ Branch Deployments |
| AI triage hook | Sidecar service consuming Argo Events | Native `@failure_hook` |
| Testing | Requires k8s cluster (kind/minikube) | Plain `pytest` |
| Jobtracker reuse | Rewrite dashboards against artifact URIs | Dashboards call Dagster GraphQL; re-use most of tracker UI |
| Team learning curve | YAML + k8s primitives | Python + typed resources |

### 2.5 When Argo would be the right call

- The jobs were pure container steps with **no customer-facing
  metadata story**
- The workload were **heavily polyglot** (Python + Go + Rust + …)
- The team lived in **GitOps-first YAML** (Argo CD already in place)
- **Kubeflow-style ML pipelines** were the main workload
- CI-as-k8s (running builds as workflows) was a primary use case

Blitzy matches none of these strongly enough to flip the decision.

---

## 3. Ranked for Blitzy — full reasoning

### 3.1 Dagster (1st)

Why it wins:

- **Asset model matches our outputs.** tech-spec, code-graph,
  repo-mapping, dependency-map, reverse-code PR — these are the
  nouns of the business. Asset-centric orchestration maps 1:1.
- **Typed metadata is the billing story.** `files_touched`,
  `lines_added`, `hours_saved`, `percent_complete`, `tokens_input`,
  `cost_usd` all become `MetadataValue` entries on the run — no
  parallel metrics pipeline.
- **Python-only is not a constraint** — all seven jobs are Python.
- **Zero-infra local dev** — the replay loop in
  `local-dev-plan.md` works out of the box.
- **Branch Deployments** — Dagster+ only. Under self-hosted OSS (our
  working assumption — see `open-questions.md`), Branch Deployments
  drop out, but equivalent functionality can be recreated on OSS
  Dagster. Recommended approach: shared `dagster-review` instance with
  code-location-keyed resource isolation per PR. See
  `branch-deployments-oss.md` for four approaches (A–D) and the
  recommendation.
- **Native `@failure_hook`** — the AI-triage pattern from
  `ai-integration-options.md` plugs in with one decorator.
- **Tracker reuse** — Dagster's GraphQL API replaces ~30% of the
  tracker's REST endpoints (job lifecycle) and **deletes** another ~5%
  (event ingestion goes away entirely). ~50% stay on the tracker DB
  (business-entity BI queries that Dagster has no model for). The
  original "~40%" figure was a rough estimate; the verified
  endpoint-by-endpoint breakdown is in `tracker-graphql-mapping.md`.

### 3.2 Airflow 3 (2nd)

Why it ranks here:

- **Asset scheduling + Task SDK** closed the biggest pre-3.0 gaps
- **Largest operator catalog** in OSS — but we don't need 80 of them
- **Multi-language support** is a real differentiator, but only if
  we add non-Python jobs. Today: zero such jobs.
- **React UI rewrite** brings UI parity with Dagster on navigation
- **Managed options** (Astronomer, MWAA, GCP Composer) are mature

Why it doesn't win:

- Still requires scheduler + DB + webserver for any real work
- Typed I/O / I/O managers are not on the roadmap
- Branch Deployments remain a DIY exercise
- Our tracker rebuild is less clean against Airflow's REST API than
  Dagster's GraphQL
- No equivalent to Dagster's `MetadataValue`-rich UI

### 3.3 Argo Workflows (3rd)

Ranks last **for Blitzy's workload** — not because it's a weaker
tool.

- Different category: container pipeline engine, not data orchestrator
- Forces us to rebuild exactly the metadata + UI layer we're trying
  to simplify in the tracker work
- Lightest ops footprint, but the ops savings don't offset the
  engineering rebuild cost

If the team ever decides that all customer-facing metrics should live
in a separate analytics service and the orchestrator should just move
bytes, Argo becomes interesting again.

> **Caveat (2026-05-21):** the ranking above assumed Dagster's ops footprint
> was acceptable. If the orchestrator must be shipped to **BlackBox /
> customer-hosted deployments**, Argo Workflows' lightest-footprint advantage
> (controller pod + CRDs) becomes much more compelling than Dagster's
> control-plane-plus-Postgres footprint. In that scenario Argo Workflows can
> plausibly outrank Dagster — at the cost of the metadata/lineage features
> that drove the original Dagster pick. See `open-questions.md` for the
> BlackBox-portability decision; the ranking should be re-evaluated once
> that's resolved.

---

## 4. What could change the ranking

| Trigger | New #1 |
|---|---|
| Blitzy onboards non-Python jobs (Go, Java, R) | Airflow 3 gains major ground; re-evaluate |
| Team adopts strict GitOps + Argo CD for infra | Argo gains; still behind Dagster on metadata |
| Dagster+ pricing becomes unworkable and OSS lacks Branch Deployments | Airflow 3 gains (managed services mature) |
| We decide the tracker owns all metrics and the engine only orchestrates | Argo becomes competitive |
| Airflow 3.x ships typed I/O equivalent to Dagster I/O managers | Gap closes further — re-evaluate |

Recommended revisit cadence: **12 months**, with a lightweight
re-comparison if any of the above trigger conditions fires.

---

## 5. Open questions

- Do we want to run a small Airflow 3 PoC alongside the Dagster PoC
  to validate the ranking, or trust the analysis?
- What's the OKR we care most about — developer velocity,
  operational simplicity, or feature-rich observability? The ranking
  holds in all three but the margin differs.
- If Dagster+ is ruled out, do Branch Deployments still matter enough
  to keep Dagster first? (See PLAN.md §4 — the multi-env win is
  weaker on OSS Dagster than on Dagster+.)
