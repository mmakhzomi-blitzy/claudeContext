# Running Blitzy jobs through Dagster — Plan & Evaluation

**Prepared:** 2026-04-20
**Author:** mohammed@blitzy.com
**Scope:** seven `archie-job-*` services under `/Users/m.makhzomi/Projects/blitzy-platform/`

---

## 1. Current state of Blitzy jobs

### 1.1 Jobs in scope

| Job | Trigger / Next step | Execution surface |
|---|---|---|
| `archie-job-code-downloader` | Emits to `GRAPH_CODE_TOPIC` or submits a k8s Job to `code-graph-generator` per batch | GCP Cloud Run Job |
| `archie-job-code-graph-generator` | Emits `reverse_document` trigger to `reverse-file-mapper` when all batches finish | GCP Cloud Run Job / k8s |
| `archie-job-document-generator` | Emits to `PLATFORM_EVENTS_TOPIC` with tech-spec phase | GCP Cloud Run Job |
| `archie-job-reverse-file-mapper` | Publishes to `GENERATE_REVERSE_THINKING_TOPIC` | GCP Cloud Run Job |
| `archie-job-reverse-thinking-generator` | Submits k8s Job (`REVERSE_CODE_GENERATOR` / `_WINDOWS`) based on target OS | GCP Cloud Run Job → k8s |
| `archie-job-reverse-code-generator` | Emits completion notifications with billing report | GCP Cloud Run Job / k8s |
| `archie-job-code-generator` | Emits to `UPLOAD_CODE_TOPIC` | GCP Cloud Run Job |

### 1.2 Observations (from code)

- Every job is a `python main.py` process that reads an **`EVENT_DATA` env var** (JSON payload), runs a **LangGraph `StateGraph`**, and emits Pub/Sub notifications.
- Chaining is **implicit**: each job calls `publish_notification(...)` or `submit_kubernetes_job(...)` to start the next stage. There is **no central DAG** — topology lives in the code.
- Observability is split across: Pub/Sub events (IN_PROGRESS / DONE / FAILED), **LangSmith tracing**, and custom notifier fields (`files_touched`, `lines_added`, `lines_edited`, `lines_removed`, `hours_saved`, `percent_complete`, `pr_data`). A `BillingReport` is uploaded to GCS at the end of `reverse-code-generator`.
- Deployment: each repo has its own `.github/workflows/deploy-job.yml` that pins `environment: dev`, reads `_DEV` vars/secrets only, and runs `gcloud run jobs deploy`. **QA and staging are not currently wired.**
- Retries: Cloud Run Jobs `--max-retries 0` (down to the job to handle); graph ingestion uses `archie_exponential_retry` in code.

---

## 2. Why a workflow engine (Dagster) adds value here

### 2.1 The concrete pains a DAG engine solves

1. **No single source of truth for the pipeline.** Today the order `downloader → graph → file-mapper → reverse-thinker → reverse-code` is distributed across seven `publish_notification` / `submit_kubernetes_job` calls. A Dagster job (or asset graph) makes it explicit, diffable, and testable.
2. **No cross-job run tracking.** A "project build" spans 3–5 jobs and fan-outs by batch. There is no single run-id that stitches the whole thing together — status has to be reconstructed from Pub/Sub events. A workflow engine gives you one run with parent/child steps.
3. **Retries and back-off live in each job.** Exponential-retry is re-implemented per repo; Dagster provides `RetryPolicy` once, and per-op overrides.
4. **Local reproduction is hard.** You need EVENT_DATA, GCP creds, Pub/Sub topics, Neo4j, and LangSmith before a job will import. `dagster dev` lets you load the pipeline without production side-effects.
5. **Resource coupling.** Each job re-creates `PublisherClient`, `storage.Client`, `CodeGraphBuilder`, `AdminStorageService`. These become typed **Dagster resources** instantiated once per run.

### 2.2 Built-in operations you'd inherit

Airflow pioneered the operator catalog — Dagster provides the same capabilities through **ops + integration packages** (`dagster-k8s`, `dagster-gcp`, `dagster-dbt`, `dagster-airbyte`, `dagster-aws`, `dagster-snowflake`, `dagster-pandas`, `dagster-slack`, etc.):

| Capability | Airflow | Dagster |
|---|---|---|
| Run a shell command | `BashOperator` | `shell_op` / `shell_command_op` (`dagster-shell`) |
| Launch a k8s Pod | `KubernetesPodOperator` | `k8s_job_op` (`dagster-k8s`) — the direct analogue, plus `K8sRunLauncher` for the whole run |
| Talk to a SQL DB | `PostgresOperator`, `MySqlOperator`, many hooks | Integration resources (`dagster-postgres`, `dagster-snowflake`, `dagster-bigquery`) + Python I/O managers |
| Trigger a dbt run | `DbtRunOperator` | `dagster-dbt` (first-class asset integration) |
| HTTP / REST call | `SimpleHttpOperator` | Plain Python op + typed resource |
| Sensors (file, Pub/Sub, GCS) | Many built-in sensors | `@sensor` decorator + per-integration helpers |
| Branching | `BranchPythonOperator` | Conditional asset materialization / dynamic outputs |

**Key difference:** Airflow has a larger pre-built operator catalog (many niche systems have a ready operator). Dagster has fewer "named operators" but closes most of the gap via its **resources + I/O manager** pattern — you write typed Python, not YAML glue.

For the blitzy jobs specifically, the migration mostly uses:
- `k8s_job_op` (to run the existing Docker images unchanged)
- Custom `@op` wrappers (to preserve the Python entrypoint while Dagster owns orchestration)
- `dagster-gcp` resources for Pub/Sub and GCS
- Custom LangSmith resource

### 2.3 Job tracking & visibility

What you get out-of-the-box from the Dagster UI:

- **Runs page** — every trigger produces a run with status, duration, and a gantt view of ops.
- **Asset graph** — if you model outputs (tech-spec, repo-mapping, dependency-map, reverse-code PR) as **assets**, the UI draws the lineage and shows which asset is stale.
- **Per-op logs** streamed in the UI with structured metadata (file path, line counts, token counts, LLM model version) via `context.log` and `MetadataValue`.
- **Run retries** — one click to re-run from a failed step, preserving upstream results.
- **Backfills** — re-materialize a range of assets (e.g. re-ingest a company's repos) with run grouping.
- **Timeline view** across runs — see "document-generator took 45m yesterday vs 12m today."

### 2.4 Metrics — what Dagster captures automatically

| Metric | Source | Visible where |
|---|---|---|
| Step status, duration, start/end | Dagster run events | UI run page, exported via `dagster_event_log` to Postgres/OSS |
| Retry counts, failures, error classes | Same | UI + metrics export |
| Asset materialization count & freshness | Asset events | Asset catalog + freshness policies |
| Step-level CPU/mem (when on k8s) | `K8sRunLauncher` + Prometheus scrape | Prometheus/Grafana |
| Tag-based filters (env, company, job_type) | Run tags | UI + queries |

### 2.5 Custom metrics you already care about

Everything under `BillingReport` and the reverse-code notifier can be surfaced as first-class Dagster metadata — no extra infrastructure required:

```python
from dagster import op, OpExecutionContext, MetadataValue, Output

@op
def emit_billing(context: OpExecutionContext, billing: dict):
    context.log.info("Reverse code generation complete")
    context.add_output_metadata({
        "files_touched":   MetadataValue.int(billing["files_modified"]),
        "lines_added":     MetadataValue.int(billing["breakdown"]["additions"]),
        "lines_edited":    MetadataValue.int(billing["breakdown"]["edits"]),
        "lines_removed":   MetadataValue.int(billing["breakdown"]["removals"]),
        "percent_LOC":     MetadataValue.float(billing["percent_loc"]),
        "hours_saved":     MetadataValue.float(billing["hours_saved"]),
        "tokens_input":    MetadataValue.int(billing["tokens"]["input"]),
        "tokens_output":   MetadataValue.int(billing["tokens"]["output"]),
        "cost_usd":        MetadataValue.float(billing["cost_usd"]),
        "langsmith_run":   MetadataValue.url(billing["langsmith_run_url"]),
    })
```

These show up directly on the run & asset pages, are queryable via GraphQL, and can be exported to Datadog / Prometheus via `dagster-prometheus` or a custom sensor. LangSmith token counts can be pulled via its SDK inside an op and attached the same way — no parallel metrics pipeline needed.

---

## 3. Dagster vs Airflow 3 — side-by-side

Airflow 3.0 (GA April 2025) closed a meaningful chunk of the historical
Dagster lead — specifically on asset scheduling, multi-language task
execution, and the UI. The Airflow rows below reflect 3.x, not 2.x.

| Dimension | Dagster | Airflow 3 |
|---|---|---|
| **Core abstraction** | Software-defined **assets** + ops | Tasks + **Assets (new in 3.0)** |
| **Language** | Python only, typed resources & I/O managers | Python + **Task SDK**: Go (GA), Java / R (roadmap) |
| **Local dev** | `dagster dev` runs the full UI+scheduler locally with zero infra | Still requires DB, scheduler, webserver |
| **Testing** | Ops and resources are plain Python; unit-test without a scheduler | DAGs need a test harness; **stable DAG authoring API (3.0)** makes this less painful |
| **Operator catalog** | Smaller pre-built catalog, but strong typed integrations | Largest catalog (airflow-providers ships 80+ systems) |
| **Remote execution** | `K8sRunLauncher` + `k8s_job_executor` + `k8s_job_op` | **Edge Executor + Task SDK** — tasks run anywhere, isolated from scheduler (3.0) |
| **Data lineage** | Native asset graph with column-level lineage (Dagster+) | Assets are new in 3.0; lineage UI still maturing; OpenLineage integration remains the richer path |
| **Observability** | Structured metadata on ops/assets, built-in UI | Improved logs; **React UI rewrite (3.0)** |
| **Typing / I/O** | Typed inputs/outputs, pluggable I/O managers (pass huge dataframes cleanly) | Xcom still primary for small data; large artifacts handled by custom operators |
| **Scheduling** | Cron + sensors + freshness policies + auto-materialize | Cron + sensors + **data-aware asset scheduling (3.0)** |
| **DAG versioning** | Code locations, hot reload | **Native DAG versioning (3.0)** — git-like history |
| **Branch/PR envs** | Dagster+ **Branch Deployments** (ephemeral per-PR) | Not built-in |
| **SaaS** | Dagster+ (Cloud / Hybrid) | Astronomer, MWAA, Cloud Composer |
| **Community size** | Growing; narrower connector catalog | Largest in OSS orchestration |
| **Best fit** | Data/AI platforms where outputs matter, strong typing preferred | Shops with many vendor systems, non-Python workloads, or existing Airflow muscle memory |

**Applied to Blitzy:** the pipeline is a small (7-node) graph where the *outputs* (tech-spec, code-graph batches, reverse-code PR, billing report) are what stakeholders actually care about. All seven jobs are Python, so Airflow 3's multi-language win doesn't apply. The connector breadth of Airflow is not a strong pull since the existing jobs talk to GCP Pub/Sub, GCS, Neo4j, GitHub — all of which have idiomatic Dagster resources or trivial Python clients. Dagster is still the better fit, with a narrower margin than pre-3.0.

## 3b. Argo Workflows — a third option

Argo Workflows is often grouped with Dagster/Airflow, but it's a
different category of tool: a **Kubernetes-native workflow engine**
where every step is a container Pod, workflows are YAML CRDs, and the
data model is "files and artifact URIs."

### Where Argo wins

- Purely k8s-native — no control plane beyond a lightweight controller
- Language-agnostic — any container is a step
- Massive parallelism (thousands of concurrent workflows in production)
- GitOps-friendly (YAML + Argo CD)
- CNCF graduated; adopted at BlackRock, Intuit, Red Hat
- Argo Events for rich event-driven triggers (separate CNCF project)

### Gaps for Blitzy specifically

- **No assets, no lineage, no data-awareness** — the concepts don't exist in the model
- **No typed I/O** — outputs are artifact-repo URIs; we'd rebuild the metadata layer we're trying to simplify
- **YAML-heavy** — refactoring seven chained workflows is painful vs Python
- **Minimal UI metadata** — Gantt + Pod logs only; no equivalent to Dagster `MetadataValue` for tokens / %LOC / cost
- **No Python-native testing** — requires a k8s cluster for meaningful tests
- **No hooks primitive** — the AI-triage pattern would need a sidecar microservice
- **Smaller active contributor base (2024-25)** than Dagster or Airflow — Argo Workflows fell into a "warning zone" per the 2025 orchestration ecosystem report

### The trade-off applied to Blitzy

What Argo buys us: a lighter operational footprint. Every existing
`archie-job-*` image becomes a step with zero wrapper code.

What Argo costs us:

- The **billing metadata story** — no first-class metadata; we'd
  rebuild the tracker's dashboards against artifact URIs
- **Asset lineage** (tech-spec → code-graph → PR) would live in
  external docs, not the engine
- **LangSmith / token / %LOC observability** — Argo has no equivalent
- **The AI-triage hook pattern** — requires a sidecar per workflow
- **Sales & Engineer dashboards** — map naturally to Dagster's GraphQL
  API; Argo's API surface is thinner

Argo would win if Blitzy's jobs were pure container pipelines with no
customer-facing metadata story. They're not — the `BillingReport` and
`%LOC` fields are exactly the value the platform surfaces to users.

## 3c. Ranked for Blitzy

| # | Tool | Rationale |
|---|---|---|
| 1 | **Dagster** | Asset model + typed metadata match the billing / %LOC signals; Python-only jobs; fast local dev; Branch Deployments for multi-env |
| 2 | **Airflow 3** | Closed much of the gap with assets + Task SDK; largest operator catalog; only beats Dagster if we need non-Python tasks (we don't today) |
| 3 | **Argo Workflows** | Great k8s pipeline engine, but weak data primitives; we'd rebuild the metadata/UI layer we're trying to simplify — net negative for Blitzy |

The prior intuition (Dagster > Airflow > Argo) is **correct for this
workload**. If Blitzy were a pure-container ML platform, the order
would flip to Argo > Airflow > Dagster — but it isn't.

**One caveat:** if Airflow 3.0's Task SDK matures into full Java / R /
other language support and we start onboarding non-Python jobs, the
gap with Dagster narrows materially. Revisit in 12 months.

---

## 4. "Dagster is easier to deploy across dev/QA/stage" — investigation

### 4.1 The claim

Dagster is easier to run the same code against dev, QA, and stage environments than Airflow.

### 4.2 What's true

- **Resources are the env boundary.** In Dagster, environment-specific config (Pub/Sub topics, GCS buckets, secrets, Neo4j hosts) lives in a **resource definition** picked per `Definitions` (e.g. `defs_dev.py`, `defs_qa.py`). Ops are env-agnostic. In Airflow, env-specific values are typically scattered across Connections, Variables, and operator kwargs — same code can still hit prod values by accident.
- **Code locations** let you run the *same* Python package against *different* `Definitions` per env without forking. `workspace.yaml` references module paths; env decides which `Definitions` to load.
- **Dagster+ Branch Deployments** spin up a full, isolated Dagster deployment per PR, pointed at a dev-only cluster — closest to "preview envs" you'll find in OSS orchestration. Airflow has no native equivalent; teams typically stand up parallel Airflow instances themselves.
- **Local = prod topology.** `dagster dev` uses the same `Definitions`, just with a dev resource set. The blitzy jobs today require GCP creds to import — Dagster flips that with in-process / mock resources.
- **CI/CD is first-class in Dagster+.** The GitHub Action validates code locations, diffs assets, and deploys to the matching env. Today each blitzy repo has its own deploy-job workflow; one pipeline replaces seven.

### 4.3 Where the claim needs qualifiers

- **"Easier" depends on starting point.** If an org already runs Airflow with a mature multi-env setup (Astronomer Deployments, MWAA per env, or Helm per namespace), Dagster isn't dramatically easier — both work.
- **Dagster+ vs OSS Dagster.** Branch Deployments are a **Dagster+** feature. If you stay on OSS Dagster, you get parity with "config-driven Airflow" — better, but not a step-change.
- **Secrets management is identical.** You still need Secret Manager / Vault / k8s Secrets. The engine doesn't remove that work.
- **Workload isolation is still your job.** Running Dagster QA and Dagster stage in the same cluster requires namespace / IAM discipline; same as Airflow.
- **Migration cost.** Adopting Dagster means introducing a new control plane. If the current GitHub-Actions-to-Cloud-Run flow is adequate, the incremental win on multi-env is real but not urgent.

### 4.4 Verdict

**The claim is directionally correct but not absolute.** Dagster's resource model, code-location workspace, and (on Dagster+) branch deployments make multi-env the *default* path, whereas Airflow tends to require deliberate engineering to get the same cleanliness. For the blitzy jobs specifically — where dev is the only wired environment today — adopting Dagster would bring QA and stage online with substantially less glue than replicating seven GitHub Actions workflows per env.

---

## 5. Other data/workflow options to consider

The three main contenders (**Dagster**, **Airflow 3**, **Argo**) are
covered above. The table below rounds out the landscape.

| Tool | Model | When to pick it |
|---|---|---|
| **Prefect (3.x)** | Pythonic flows, dynamic tasks, managed Cloud | Strong when orchestration is mostly Python and you want a managed control plane |
| **Temporal** | Durable workflows, code-first state | Not a data-orchestrator per se — fits long-running, stateful workflows (e.g. a user's project lifecycle); complements rather than replaces Dagster |
| **Kestra** | YAML-first, event-driven | JVM/YAML shops; pluggable and event-native |
| **Flyte** | Strongly-typed Python, k8s-native | ML pipelines with heavy caching + versioning needs |
| **AWS Step Functions / GCP Workflows** | Cloud-managed state machines | All-in on one cloud; small DAGs; no separate control plane to run |
| **Luigi** | Python tasks with dependencies | Legacy choice; simple batch pipelines; mostly superseded |
| **Cloud Composer / MWAA / Astronomer** | Managed Airflow | Want Airflow without running it |

For the Blitzy workload (≤10 nodes, GCP-native, Python heavy, LLM-token metrics): **Dagster > Airflow 3 > Argo** on the shortlist (see section 3c). Prefect and Step Functions are credible alternatives. Temporal is a complement, not a replacement.

---

## 6. Phased implementation plan

Start with `archie-job-code-downloader` — it's the first link in the chain,
has the simplest notification schema, no LLM dependencies, and emits the
batch indices that the rest of the graph fans out on. Getting it right
de-risks everything downstream.

Timeline assumes one engineer at ~60% allocation. Each phase ends with a
demo-able milestone. Phase gates are marked with ✅.

### Phase 1 — Foundation + `code-downloader` PoC · Weeks 1–2

**Goal:** one working Dagster job that runs `archie-job-code-downloader`
end-to-end, triggered locally, with metadata on the run page.

**Tasks**

1. Bootstrap a Dagster repo (`blitzy-orchestration/`): `dagster`,
   `dagster-webserver`, `dagster-k8s`, `dagster-gcp`, Python 3.12.
2. Define a skeleton `Definitions` module with one resource per concern:
   `PubSubResource`, `StorageResource` (wraps `AdminStorageService`),
   `LangSmithResource`. For this phase, `StorageResource` is a thin
   pass-through — no refactor of the job yet.
3. Wrap `archie-job-code-downloader` as a `k8s_job_op` that runs the
   existing Docker image unchanged. Env vars come from the resource's
   `build_env_vars()` method so dev/QA/stage can swap topic names without
   editing op code.
4. Run locally via `dagster dev` against the dev GCP project. Feed a
   sample `EVENT_DATA` (copy one from a real run) via Dagster run config.
5. Attach op metadata to the run after completion by parsing the
   Pub/Sub `DONE` event emitted by the existing job (read-only — no
   change to the job yet): `lines_onboarded`, `files_onboarded`,
   `total_files`, `file_extensions`, `batch_indexes`.
6. Add a GitHub Action that validates the Dagster code location
   (`dagster-cloud ci check` or OSS equivalent) on every PR.

**Deliverables**

- ✅ One Dagster run that successfully executes `code-downloader`
  against dev and shows per-run metadata
- ✅ Decision: Dagster+ (for Branch Deployments) vs OSS Dagster
- ✅ PoC repo in `blitzy-orchestration` with CI on PRs

**Risks**

- Cloud Run→k8s parity: the job's existing `gcloud run jobs deploy`
  sets `--task-timeout 24h` and VPC/egress flags. Phase 1 k8s Pod needs
  matching limits. Target: same region, same VPC, same SA.
- Pub/Sub subscriber contention: existing subscribers on
  `PLATFORM_EVENTS_TOPIC` keep consuming; Dagster just observes.

---

### Phase 2 — `code-graph-generator` + batch fan-out · Weeks 3–4

**Goal:** one Dagster run spans download + graph batches, with the
fan-out/fan-in visible in the UI.

**Tasks**

1. Wrap `archie-job-code-graph-generator` as a `k8s_job_op`.
2. Convert the downloader's output (`batch_indexes` + `total_batches`)
   into **dynamic outputs**; each batch becomes one materialization of
   the graph op, running as its own Pod.
3. Replace the in-code `graph_builder.are_other_batches_complete(...)`
   batch-done check with Dagster's native fan-in. Keep the Neo4j check
   as a defensive guard behind a feature flag.
4. Add a `@sensor` watching the existing `GRAPH_CODE_TOPIC` — when an
   external trigger lands, launch the combined Dagster run. This lets
   the rest of the platform keep publishing Pub/Sub events unchanged
   during the transition.
5. Use Dagster `RetryPolicy(max_retries=3, delay=60, backoff=EXP)` on
   the graph op to start retiring `@archie_exponential_retry`.

**Deliverables**

- ✅ Downloader → per-batch graph fan-out → aggregation in one run
- ✅ Sensor-triggered runs from Pub/Sub working in dev
- ✅ Retry policy observed in UI (re-run counts, retry delays)

**Risks**

- Dynamic output cardinality: very large repos can push hundreds of
  batches. Set `max_concurrent` on the executor so we don't stampede
  GKE.
- Neo4j contention across parallel batches — same risk as today; not
  made worse by Dagster but worth re-measuring.

---

### Phase 3 — Reverse chain (file-mapper → thinker → code) · Weeks 5–6

**Goal:** full reverse pipeline runs under one Dagster run and surfaces
the `BillingReport` as first-class metadata.

**Tasks**

1. Wrap the three remaining reverse jobs as `k8s_job_op`s.
2. Handle the per-OS fan-out in `reverse-thinking-generator`: choose
   **option A** (keep internal k8s submit, Dagster treats it as a leaf)
   for Phase 3, with a TODO to move to Dagster dynamic outputs in
   Phase 5. This keeps the Windows/Linux logic intact.
3. In `reverse-code`, read the `BillingReport` back from GCS after the
   Pod finishes and attach every field as `MetadataValue`
   (`files_touched`, `lines_{added,edited,removed}`, `hours_saved`,
   `percent_complete`, token counts, cost_usd, LangSmith run URL).
4. Promote these outputs to **assets** in the UI:
   `tech_spec`, `code_graph`, `repo_mapping`, `dependency_map`,
   `sorted_files`, `reverse_code_pr`. Freshness policies can wait.
5. Add a dashboard (simple Grafana panel driven by the Dagster
   Postgres event log) showing lines-per-hour, tokens-per-PR,
   failures-by-stage.

**Deliverables**

- ✅ End-to-end reverse build in a single Dagster run
- ✅ Billing metadata visible on the run and on each asset
- ✅ First multi-stage Grafana panel

**Risks**

- Long runs (some reverse-code runs are multi-hour). Dagster
  run-monitor timeout must align with k8s Pod activeDeadlineSeconds.
- `BillingReport` schema has only been *inferred* — confirm with the
  team owning `reverse-code` before relying on field names.

---

### Phase 4 — Independent jobs (`document-generator`, `code-generator`) · Week 7

**Goal:** every `archie-job-*` has a Dagster entrypoint, even the ones
not in the main chain.

**Tasks**

1. Wrap `archie-job-document-generator` — standalone job, no children.
2. Wrap `archie-job-code-generator` — separate pipeline with its own
   upload topic.
3. Add schedules / sensors per job type.
4. Consolidate shared resources (`LangSmithResource`, LLM config) into
   a single `common_resources.py`.

**Deliverable**

- ✅ All seven jobs runnable via Dagster; feature parity with today's
  Pub/Sub-triggered flow.

---

### Phase 5 — Metadata + native-op polish · Week 8

**Goal:** graduate at least one job from `k8s_job_op` wrapper to a
native Dagster `@op` so we validate the resource-injection pattern.

**Tasks**

1. Pick `archie-job-code-generator` (simplest — no k8s self-fan-out,
   no LangGraph recursion surprises) and refactor `main.py` so its
   logic is a plain Python function that accepts resources as args.
   Module-level `os.environ[...]` reads go away.
2. Replace its `k8s_job_op` with an `@op` running inside the Dagster
   Pod (or configure per-op k8s executor if we still want isolation).
3. Validate: memory footprint, latency, retry behaviour.
4. If successful, convert `reverse-code` next (highest metrics value).

**Risks**

- Refactor breaks existing deploy contract. Keep the old
  `main.py` + Dockerfile available as a rollback for ≥2 weeks.
- Credential plumbing: `CodeGraphBuilder` fetches per-company Neo4j
  creds at runtime — must stay that way in the native-op version.

---

### Phase 6 — Multi-env (QA, stage) wiring · Weeks 9–10

**Goal:** the claim we're testing — same code, three envs, low effort.

**Tasks**

1. Split `Definitions` into `defs_dev.py`, `defs_qa.py`, `defs_stage.py`.
   Each picks a different resource bundle (different topics, buckets,
   service URLs, Neo4j host, LLM API keys). Ops are unchanged.
2. Provision Pub/Sub topics, GCS buckets, IAM, and Secret Manager
   entries for QA and stage. (This is the work the current
   `.github/workflows/deploy-job.yml` files don't do — it has to happen
   *somewhere* regardless of engine.)
3. If on Dagster+: enable Branch Deployments pointed at dev infra.
   Each PR gets an ephemeral Dagster deployment to validate against
   real topics without affecting prod.
4. Kill the seven per-repo GitHub Actions workflows. Replace with one
   `deploy.yaml` in `blitzy-orchestration` that deploys all code
   locations per env.

**Deliverables**

- ✅ Same job runnable in dev / QA / stage with only config diff
- ✅ (Dagster+) Per-PR preview env working
- ✅ Single deploy pipeline replaces seven

**Risks**

- Biggest infra lift of any phase — but much of it is infra work we'd
  need to do to wire QA/stage even if we stayed on Cloud Run.
- IAM sprawl: service accounts multiply by 3 envs. Use IaC (Terraform
  module per env) from day one.

---

### Phase 7 — Cutover and decommission · Week 11+

**Goal:** retire the per-repo deploy workflows and make Dagster the
only orchestrator.

**Tasks**

1. Two-week dual-run period: Dagster and the existing Pub/Sub chain
   active simultaneously, publishing to different topics, compared
   side-by-side on correctness and latency.
2. Flip traffic: the upstream services that emit `EVENT_DATA` point
   at Dagster's sensor topics. Old Pub/Sub chain becomes passive.
3. Archive the per-repo `.github/workflows/deploy-job.yml` files.
4. Remove `@archie_exponential_retry`, module-level Pub/Sub client
   initialisation, and `submit_kubernetes_job` calls where replaced.
5. Document the new on-call runbook.

**Deliverable**

- ✅ Dagster is the only orchestrator for `archie-job-*` runs.

---

### Scope kept out on purpose

- Migrating LangSmith to a different tracing backend (out of scope).
- Rewriting LangGraph state machines inside each job (out of scope —
  Dagster orchestrates *between* jobs, not inside them).
- Replacing Pub/Sub for external subscribers (they keep consuming
  `PLATFORM_EVENTS_TOPIC` unchanged).

### Phase map at a glance

| Phase | Weeks | Covers | Key deliverable |
|---|---|---|---|
| 1 | 1–2 | `code-downloader` only | First Dagster run in dev |
| 2 | 3–4 | + `code-graph-generator` | Batch fan-out in one run |
| 3 | 5–6 | + reverse chain | Billing metadata live |
| 4 | 7 | + `document-generator`, `code-generator` | All jobs reachable |
| 5 | 8 | Native-op refactor of one job | Resource-injection validated |
| 6 | 9–10 | QA + stage + Branch Deployments | Multi-env working |
| 7 | 11+ | Cutover | Old deploy workflows retired |

### Risks that span all phases

- Long-running jobs (task-timeout 24h on Cloud Run). k8s Pod budgets
  and Dagster run-monitor timeouts must align.
- Two services self-submit k8s Jobs mid-run (`reverse-thinking` and
  `code-downloader`). Phase 3/5 decides whether that internal fan-out
  moves into the Dagster graph.
- Cost of running the Dagster control plane (daemon + webserver +
  Postgres) vs current "one Cloud Run Job per service" setup. Bring a
  rough monthly number to the Phase 1 review.
- LangSmith tracing stays in place — still called from inside each op.
  No compliance change expected.

---

## 7. Next steps

1. **Approve the phased plan in section 6.** The Phase 1 scope is
   deliberately tight — `code-downloader` only — so we can get real
   signal inside two weeks before committing to the full migration.
2. **Pick a PoC owner** and open the `blitzy-orchestration` repo.
3. **Decide Dagster+ vs OSS Dagster** before Phase 1 ends (Branch
   Deployments are the biggest multi-env lever and are Dagster+-only).
4. **Phase 1 exit review** — demo against dev, then decide whether to
   continue into Phase 2 or revisit the approach.

---

## Appendix A — Current per-job env contract (from `main.py` imports)

| Env var | Used by |
|---|---|
| `EVENT_DATA` | every job (JSON payload) |
| `PROJECT_ID`, `GCS_BUCKET_NAME`, `BLOB_NAME` / `PRIVATE_BLOB_NAME` | every job |
| `PLATFORM_EVENTS_TOPIC` | every job (status events) |
| `GRAPH_CODE_TOPIC` | downloader, code-graph-generator |
| `GENERATE_DOCUMENT_TOPIC` | document-generator |
| `GENERATE_REVERSE_THINKING_TOPIC` | reverse-file-mapper |
| `GENERATE_REVERSE_CODE_TOPIC` | reverse-code-generator |
| `GENERATE_REVERSE_DOCUMENT_TOPIC` | code-graph-generator |
| `UPLOAD_CODE_TOPIC`, `GENERATE_CODE_TOPIC` | code-generator |
| `NEO4J_SERVER / USERNAME / PASSWORD` | downloader, graph, file-mapper, reverse-thinker, reverse-code |
| `GITHUB_SECRET_SERVER` | downloader, graph, file-mapper, reverse-thinker, reverse-code |
| `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GOOGLE_API_KEY`, `VOYAGE_API_KEY` | LLM-using jobs |
| `LANGSMITH_*` / `LANGCHAIN_*` | all LLM jobs |
| `TOKENIZERS_PARALLELISM` | LLM jobs |

## Appendix B — Current notifier metadata fields (extractable as Dagster metadata)

- `files_touched`, `lines_added`, `lines_edited`, `lines_removed`, `hours_saved`, `percent_complete`
- `lines_onboarded`, `files_onboarded`, `total_files`, `file_extensions` (downloader)
- `total_files_processed`, `total_lines_processed` (code-graph)
- `estimated_lines_generated`, `estimated_hours_saved` (document-generator)
- `pr_data` (reverse-code)
- `BillingReport` on GCS: breakdowns, token usage (via LangSmith)
