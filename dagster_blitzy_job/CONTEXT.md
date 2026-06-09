# Session context — running Blitzy jobs through Dagster

> Drop this file (and the siblings in this folder) into a fresh Claude Code session
> to resume the conversation without re-scanning the blitzy-platform repos.

## Goal

Evaluate orchestrating the seven `archie-job-*` services through **Dagster** instead
of the current Cloud Run Jobs + Pub/Sub hand-off pattern.

The deliverables live one directory up:
- `../PLAN.md` — full written plan and evaluation
- `../index.html` — reveal.js deck of the same material

## Jobs in scope (all under `/Users/m.makhzomi/Projects/blitzy-platform/`)

1. `archie-job-code-downloader` — fetches repo to GCS, triggers code-graph
2. `archie-job-code-graph-generator` — builds Neo4j code graph in batches
3. `archie-job-document-generator` — builds tech-spec docs
4. `archie-job-reverse-file-mapper` — maps repo files to tech-spec sections
5. `archie-job-reverse-thinking-generator` — plans per-file edits, fans out per-OS
6. `archie-job-reverse-code-generator` — produces code + PR + billing report
7. `archie-job-code-generator` — standalone code-gen path (upload topic)

## Current architecture facts (verified from code)

- Every job is `python main.py` reading an `EVENT_DATA` env var (JSON payload)
- Internal logic is LangGraph `StateGraph` run with `astream` / `ainvoke`
- Status events flow to `PLATFORM_EVENTS_TOPIC` with `JobStatus` IN_PROGRESS / DONE / FAILED
- Chaining happens via `publish_notification(...)` or `submit_kubernetes_job(...)`
  - **Dual-transport, gated per-worker** (verified 2026-05-21): `archie-job-code-downloader/main.py:349-367` shows `if use_k8s: submit_kubernetes_job(ArchieJobType.CODE_GRAPH_GENERATOR, ...)` *else* `publish_notification(..., GRAPH_CODE_TOPIC)`. Resource requests (`cpu`, `memory`, `ephemeral_storage`) are inlined into worker code — i.e., manual orchestration in business code, which an orchestrator would absorb.
  - **Dual-publish per completion** (verified in `archie-job-reverse-file-mapper/main.py:215-233`, same pattern in reverse-thinking-generator and code-graph-generator): on completion each worker publishes TWO messages — one to the next-stage topic (chain hop) and one to `PLATFORM_EVENTS_TOPIC` (status feed for `archie-platform-event-listener`).
  - **`archie-shared/blitzy_platform_shared/notifier.py`** centralizes the publish path and directly imports `google.cloud.pubsub_v1.PublisherClient`. This is the single cloud-coupling chokepoint shared by all workers.
- **`archie-platform-event-listener` is a projection consumer, not a dispatcher** (verified 2026-05-21): subscribes to `PLATFORM_EVENTS_TOPIC` and performs DB updates, email, PDF gen, metering, branch-lock release. No `PublisherClient` is instantiated in its `src/`; no `submit_kubernetes_job` calls. Under an orchestrator migration it either disappears (asset materialization writes DB via I/O managers) or shrinks to a thin sensor.
- Observability: LangSmith tracing (`langsmith_tracing` context manager)
- Storage: GCS via `AdminStorageService`; graph via `CodeGraphBuilder` → Neo4j
- Custom metrics already emitted (perfect candidates for Dagster metadata):
  - `files_touched`, `lines_added`, `lines_edited`, `lines_removed`
  - `hours_saved`, `percent_complete`, `pr_data`
  - `lines_onboarded`, `files_onboarded`, `file_extensions`
  - `total_files_processed`, `total_lines_processed`
  - `estimated_lines_generated`, `estimated_hours_saved`
  - `BillingReport` uploaded to GCS at end of reverse-code
- Deploy: per-repo `.github/workflows/deploy-job.yml` → `gcloud run jobs deploy`
  with `environment: dev` hardcoded and only `_DEV` vars/secrets wired.
  **QA and stage are not currently configured.** *(Note: this applies to the worker repos' deploy workflows specifically. `archie-platform-event-listener` does not have a `deploy-job.yml` at all — it uses a separate CI flow. So "only dev is wired" is not a uniform statement across the platform.)*

## Key findings from research

1. **Workflow engine wins for us:** explicit DAG, single run-id across jobs,
   typed retries, local reproduction, shared resource instances, one-click
   reruns, asset lineage, structured metadata (token counts, %LOC, etc.).
2. **Built-in ops story:** Airflow has a bigger named-operator catalog;
   Dagster closes the gap with typed Python ops + `dagster-k8s`, `dagster-gcp`,
   `dagster-shell`, `dagster-dbt`, etc. For blitzy the important ones are
   `k8s_job_op` (run existing images unchanged) and `dagster-gcp` (Pub/Sub + GCS).
3. **Dagster vs Airflow 3 vs Argo:** Ranked Dagster > Airflow 3 > Argo for
   Blitzy. Airflow 3.0 (April 2025) closed meaningful ground with asset
   scheduling + Task SDK (Go today, Java/R roadmap), but stays #2 because we
   have no non-Python workloads and no typed-I/O equivalent. Argo is a
   different category (k8s container pipelines) and would force us to rebuild
   the metadata/UI layer. Full analysis: `orchestrator-comparison.md`.
   Revisit in 12 months.
4. **Multi-env claim investigation:** *directionally true, not absolute*.
   Dagster's `Definitions` per env, code locations, and Dagster+ Branch
   Deployments make multi-env the default path. Caveats: Branch Deployments
   are Dagster+-only, secrets work is unchanged, migration has a control-plane
   cost. Starting from "only dev is wired" makes the case stronger.
5. **Shortlist of alternatives:** Dagster > Prefect > Argo > GCP Workflows /
   Step Functions. Temporal complements rather than replaces.

## Proposed minimal migration slice

Wrap each existing Docker image as a `k8s_job_op` — no change to job code.
Example:

```python
from dagster import job, Definitions
from dagster_k8s import k8s_job_op

download_code = k8s_job_op.configured({
    "image": "us-east1-docker.pkg.dev/.../archie-job-code-downloader:latest",
    "env_vars": ["EVENT_DATA", "PROJECT_ID", "GCS_BUCKET_NAME", ...],
    "container_config": {"resources": {"requests": {"cpu": "1", "memory": "16Gi"}}},
}, name="download_code")

@job
def archie_full_build():
    graph   = code_graph(download_code())
    mapping = reverse_file_map(graph)
    thinker = reverse_thinking(mapping)
    reverse_code(thinker)
```

## Open questions to pick up next session

- Do we adopt Dagster+ (for Branch Deployments) or run OSS Dagster on our own GKE?
- Keep `reverse-thinking-generator`'s internal k8s fan-out, or pull it into the
  Dagster DAG as dynamic outputs?
- What's the path for the three still-wired destinations (UPLOAD_CODE_TOPIC,
  GENERATE_REVERSE_DOCUMENT_TOPIC, PLATFORM_EVENTS_TOPIC subscribers) — do they
  stay on Pub/Sub or move to Dagster sensors?
- Sizing: how many concurrent builds should the Dagster deployment support?
- Who owns the PoC — platform team or a dedicated orchestration workstream?

## Files in this folder

- `CONTEXT.md` — you're reading it
- `job-inventory.md` — per-job facts distilled from `main.py`
- `dagster-mapping.md` — how each current job maps to Dagster primitives
- `env-contract.md` — env vars each job expects
- `phase-plan.md` — 7-phase rollout table with Phase 1 definition-of-done
- `local-dev-plan.md` — capture/replay/verify loop for iterating locally
- `initial-prompts.md` — ready-to-paste prompts to bootstrap Phase 1 in a fresh Claude Code session
- `ai-integration-options.md` — AI-augmented monitoring, notifications, auto-PR, and smart retrigger; phased AI-Phase A–E rollout
- `orchestrator-comparison.md` — Dagster vs Airflow 3 vs Argo Workflows, ranked for Blitzy, with revisit triggers
- `metadata-improvements.md` — tracker metadata audit + M1–M5 sub-phases to ship quick wins in ~4 weeks, independent of the orchestrator migration
- `tracker-graphql-mapping.md` — endpoint-by-endpoint mapping of `archie-job-tracker` REST to Dagster GraphQL; verifies the "~40%" claim in `orchestrator-comparison.md` §3.1
- `branch-deployments-oss.md` — what Dagster+ Branch Deployments are and four approaches (A–D) for recreating equivalent functionality on self-hosted OSS Dagster
- `open-questions.md` — exactly the list above, editable as the conversation evolves
