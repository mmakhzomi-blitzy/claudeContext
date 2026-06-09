# Open questions to resolve

Edit this file as the conversation progresses.

## Strategic

- [x] **Dagster+ vs OSS Dagster?** Branch Deployments (the strongest multi-env
      argument) are Dagster+-only. ~~Is the per-user cost acceptable?~~
      **Direction set (2026-05-21):** if the orchestrator must ship to
      BlackBox / customer-hosted deployments, Dagster+ is off the table
      (it's a SaaS offering). **OSS Dagster, self-hosted** is the working
      assumption. Consequence: Branch Deployments drop out of the value
      stack — re-weight `orchestrator-comparison.md` accordingly. Tracker
      rebuild can still use Dagster GraphQL since the API is identical
      between Dagster+ and OSS.
- [ ] **BlackBox orchestrator footprint** *(new, 2026-05-21)* — leaning
      "advantages outweigh the customer ops burden." But the ops footprint
      differs sharply by orchestrator: **Argo Workflows** = controller +
      CRDs (lightest); **OSS Dagster** = daemon + webserver + Postgres +
      code locations (medium); **Airflow 3** = scheduler + DB + webserver +
      executor + Redis (heaviest). If BlackBox portability is hard
      requirement, this can flip the Dagster-vs-Argo ranking. See
      `orchestrator-comparison.md` §3.3 caveat.
- [ ] **Branch Deployments strategy on OSS Dagster** *(new)* — pick one
      of Approaches A–D from `branch-deployments-oss.md`. Recommended:
      Approach D (shared `dagster-review` instance with code-location-keyed
      resource isolation per PR). Sub-decisions: resource isolation
      naming convention, CI workflow for register/deregister, cleanup
      policy on PR close, cost ceiling on concurrent open PRs.
- [ ] **Airflow 3 PoC alongside Dagster?** Run a small parallel PoC to verify
      the ranking, or trust the analysis in `orchestrator-comparison.md`?
- [ ] **12-month revisit.** Set a reminder: if Blitzy onboards non-Python jobs
      or Airflow 3.x ships typed I/O, the Dagster-vs-Airflow margin may flip.
- [ ] **Who owns the PoC?** Platform team, or a dedicated orchestration
      workstream?
- [ ] **Success criteria for the PoC?** (e.g. one full build end-to-end, with
      billing metadata visible in the UI, running against dev Pub/Sub)

## Architectural

- [ ] Keep `reverse-thinking-generator`'s internal per-OS k8s fan-out, or pull
      it into the Dagster DAG as dynamic outputs?
- [ ] Does `code-generator` share the Dagster job or live as its own job?
- [ ] Do we preserve Pub/Sub hand-off for external subscribers, or route
      everything through Dagster sensors?
- [ ] Where does `AdminStorageService` live — one shared resource or per-op?
- [ ] How do we map the existing `job_id` / `project_id` / `code_gen_id` to
      Dagster run tags for continuity with existing dashboards?

## Operational

- [ ] Sizing: how many concurrent builds does the Dagster deployment need to
      support?
- [ ] Where does the Dagster daemon + webserver run? GKE in `blitzy-os-dev` /
      Dagster+ Hybrid?
- [ ] Postgres for the event log — new instance or reuse an existing one?
- [ ] Long-running jobs: Cloud Run has 24h task-timeout. Dagster run-monitor
      timeouts must match k8s Pod budgets.
- [ ] Cost comparison: running Dagster control plane vs current seven
      Cloud Run Job services.

## Migration

- [ ] Big-bang vs per-job cutover. Minimal slice is `downloader` + `code-graph`
      together (PoC #1).
- [ ] Do we keep Cloud Run deploys as a fallback during cutover?
- [ ] How long do we run both systems side-by-side?
- [ ] Observability parity: what dashboards must exist in Dagster before we
      can retire the current Pub/Sub-based tracking?

## Investigation TODO

- [ ] Confirm GCS `BillingReport` schema (only inferred from usage)
- [ ] Check if any job has internal state that would break if wrapped as
      `k8s_job_op` (e.g. signal handlers, shutdown hooks)
- [ ] Verify that existing LangSmith traces can still be correlated to the
      new Dagster run-id (probably via run tags)
- [ ] Confirm target `dagster` version + Python compatibility (jobs pin
      Python 3.12)
- [x] **Tracker GraphQL endpoint mapping** *(verified 2026-05-21)* —
      enumerated `archie-job-tracker` route surface and mapped to Dagster
      GraphQL. Revised breakdown: ~30% replaced + ~5% deleted + ~50%
      stays on tracker DB + ~15% conditional (artifact strategy decides).
      Full table in `tracker-graphql-mapping.md`. The half-day spike to
      confirm against a working Dagster instance is documented in §6 of
      that doc — still worth doing before committing to Option B.
