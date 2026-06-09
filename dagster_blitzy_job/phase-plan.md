# Phased implementation plan — quick-reference

Full prose lives in `../PLAN.md` §6. This is the TL;DR sheet to scan
when resuming the conversation.

| Phase | Weeks | Scope | Exit criteria |
|---|---|---|---|
| 1 | 1–2 | `code-downloader` only | First Dagster run in dev; key metadata visible |
| 2 | 3–4 | + `code-graph-generator` | Batch fan-out native to Dagster |
| 3 | 5–6 | + reverse-file-mapper, reverse-thinker, reverse-code | `BillingReport` as first-class metadata |
| 4 | 7 | + `document-generator`, `code-generator` | All seven jobs reachable |
| 5 | 8 | Native-op refactor of one job | Resource-injection pattern validated |
| 6 | 9–10 | QA + stage + (Dagster+) Branch Deployments | Multi-env working |
| 7 | 11+ | Cutover | Old `deploy-job.yml` workflows retired |

## Why start with `code-downloader`

- First in the chain — downstream can't succeed if this is wrong anyway
- Simplest notification schema (no LLM tokens, no `BillingReport`)
- Emits the `batch_indexes` that the rest of the graph fans out on —
  getting fan-out right here unlocks Phase 2
- No self-submitted k8s Jobs (Phase 3's thinker has those)

## Phase 1 definition of done

1. `blitzy-orchestration/` repo exists with `Definitions` and CI
2. `k8s_job_op` wraps the current `archie-job-code-downloader:latest`
   image — no change to the job itself
3. `dagster dev` can trigger a run against the dev GCP project
4. Run metadata shows: `lines_onboarded`, `files_onboarded`,
   `total_files`, `file_extensions`, `batch_indexes`
5. Decision captured: Dagster+ (Branch Deployments) vs OSS Dagster

## What each phase does NOT cover

- Phase 1 does *not* refactor `main.py` — the job ships unchanged
- Phases 1–3 do *not* introduce native `@op` logic — everything is
  `k8s_job_op` wrappers
- Phase 6 does *not* create QA/stage GCS buckets / Pub/Sub topics —
  that's infra work we'd need to do regardless of orchestration engine;
  budget Terraform time separately

## Rollback plan

- Keep the existing GitHub Actions workflows in place until Phase 7
- Dagster runs publish to a *different* Pub/Sub topic suffix in the
  two-week dual-run; external subscribers stay on the old one
- Revert path: disable the Dagster sensor, restore Pub/Sub emission to
  the old topic, no state surgery required
