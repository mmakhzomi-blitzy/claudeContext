# Tracker REST → Dagster GraphQL mapping

> Verifies the "~40%" claim in `orchestrator-comparison.md` §3.1 against the
> actual `archie-job-tracker` route surface. Verified 2026-05-21 against
> `/Users/m.makhzomi/Projects/archie-job-tracker/src/api/routes/`.

## 1. Tracker REST endpoint inventory

| File | Endpoints | Category |
|---|---|---|
| `job.py` | ~19 | Job lifecycle, state, artifacts, trigger, bulk ops |
| `job_event.py` | 2 | Event ingestion (POST from workers) |
| `engineer_dashboard.py` | 7 | BI: summary, leaderboards, trends, quality, export |
| `sales_dashboard.py` | 8 | BI: company health, GTM metrics, projects export |
| `project.py` | 1 | `/<project_id>/loc_stats` |
| `companies.py` | 1 | Company listing |
| `auth.py` + `sso_auth.py` | several | Auth (out of scope for orchestrator) |

**Total in-scope (excluding auth): ~38 endpoints.**

## 2. What Dagster GraphQL exposes

Confirmed against Dagster GraphQL docs:

- `runsOrError(filter, limit)` — list runs, filter by status / pipeline / tag
- `runOrError(runId)` — single run + step state + tags + logs
- `logsForRun(runId)` — structured event log
- `assetsOrError` / `assetNodeOrError` — asset definitions
- `assetMaterializations(assetKey)` — materialization history with metadata
- `pipelineRunStatsOrError` — aggregate stats per pipeline
- Mutations: `launchRunMutation`, `terminateRunMutation`,
  `reloadRepositoryLocationMutation`, `deletePipelineRunMutation`

## 3. Endpoint-by-endpoint mapping

### Cleanly replaceable by Dagster GraphQL (~10–12 endpoints, ~30%)

| Tracker endpoint | Dagster GraphQL equivalent |
|---|---|
| `GET /jobs` (paginated list, filters, search, sort) | `runsOrError(filter, limit)` |
| `GET /jobs/<id>` | `runOrError(runId)` |
| `GET /jobs/stats` | `pipelineRunStatsOrError` (or computed off `runsOrError`) |
| `POST /jobs/<id>/refresh` | Reload via run tags |
| `POST /jobs/<id>/trigger` | `launchRunMutation` |
| `POST /jobs/bulk-trigger` | Loop of `launchRunMutation` calls |
| `POST /jobs/<id>/trigger-k8s` | `launchRunMutation` (k8s already underlies it via `k8s_job_op`) |
| `DELETE /jobs/<id>/delete` | `deletePipelineRunMutation` |
| `GET /jobs/metadata` | Run tags + asset metadata via GraphQL |

### Deleted entirely (~2 endpoints, ~5%)

These aren't replaced — they're *removed*, because the workers no longer
fire status events at the tracker. Dagster owns run state.

| Tracker endpoint | Why it disappears |
|---|---|
| `POST /job_events` | Workers don't post status events; Dagster authors them |
| `POST /job_events/kubernetes` | Same |

Removing these also deletes `kubernetes_event_handler.py` and
`custom_event_handler.py`.

### Conditional — depends on artifact strategy (~5–6 endpoints, ~15%)

| Tracker endpoint | Path A: pull-through GraphQL | Path B: mirror to tracker DB |
|---|---|---|
| `GET /jobs/<id>/artifacts` | `assetMaterializations(assetKey)` per call | Tracker reads its own DB |
| `GET /jobs/<id>/artifacts/availability` | Same | Same |
| `GET /project/<project_id>/loc_stats` | LOC summed from asset metadata | Cached / mirrored |

**Tradeoff:** pull-through is simpler but slower for dashboard rollups.
Mirroring keeps dashboards fast but means maintaining a Dagster sensor that
pushes asset materialization metadata into the tracker DB.

### Stays on tracker DB (~16–18 endpoints, ~50%)

Dagster has no model for engineers, companies, plan types, or subscription
billing — these are **business entities**, not runtime artifacts. Their
queries stay on the tracker's own database.

| File | Endpoints staying |
|---|---|
| `engineer_dashboard.py` | All 7 (summary, engineers, leaderboards, trends, quality, export) |
| `sales_dashboard.py` | All 8 (companies, overview, projects, exports, health, GTM, support-metrics) |
| `companies.py` | 1 |
| `job.py` | `GET /binary/latest`, `GET /jobs/<id>/setup-guide`, `GET /jobs/topics` (likely deleted, not stays — topic concept goes away under orchestrator) |

The dashboard queries can *join* Dagster-derived metadata (if mirrored) with
users/companies/plan types, but the joining and aggregation logic stays in
the tracker.

## 4. The "~40%" claim — honest revised number

| Disposition | Endpoints | Share |
|---|---|---|
| **Replaced by Dagster GraphQL** | ~10–12 | ~30% |
| **Deleted entirely** | ~2 | ~5% |
| **Conditional (artifact strategy decides)** | ~5–6 | ~15% |
| **Stays on tracker DB (BI queries)** | ~16–18 | ~50% |

So the "~40%" figure from `orchestrator-comparison.md` §3.1 is **roughly
defensible** but the framing was misleading. The clearer story:

- **~30% are GraphQL replacements** (job lifecycle reads / mutations)
- **~5% are deletions** (event ingestion goes away — a *bigger* win than replacement)
- **~50% are out of GraphQL's scope** (business-entity BI queries)
- **~15% are decided by the artifact-mirroring strategy**

## 5. What the tracker would need to do (actionable list)

1. **Add a Dagster GraphQL client** alongside the existing DB layer. Job
   lifecycle endpoints proxy queries to Dagster's GraphQL endpoint instead
   of reading `CloudRunJobTracker` rows.
2. **Delete the event-ingestion endpoints** (`POST /job_events`,
   `POST /job_events/kubernetes`) and the
   `kubernetes_event_handler.py` / `custom_event_handler.py` code.
3. **Decide on the artifact / metadata strategy** — pull-through vs mirror.
   Recommend mirror for dashboard speed; sensor in Dagster pushes asset
   materialization metadata into tracker DB.
4. **Keep the BI dashboard endpoints unchanged.** Engineer/sales dashboards
   continue to query the tracker DB, joining users/companies/plan types
   with mirrored Dagster metadata.
5. **Map run identifiers.** `CloudRunJobTracker.job_id` becomes Dagster
   `run_id` plus tags (`project_id`, `code_gen_id`, etc.). Add an
   ID-translation layer for external links and saved queries.

## 6. Half-day verification spike

To confirm the breakdown above against a working Dagster instance:

1. Stand up OSS Dagster locally with one toy job, tagged like a real run
   (`project_id`, `code_gen_id`).
2. Write the equivalent GraphQL query for
   `GET /jobs?status=RUNNING&search=...&page=2&limit=20`. Compare response
   shape and latency against the tracker's REST response.
3. Pick one dashboard query (e.g. `engineer_dashboard /summary`) and
   confirm it *cannot* be served by GraphQL — i.e. you need DB joins.
   That validates the "stays on tracker DB" half.
4. Pick one artifact query and confirm `assetMaterializations` can return
   the metadata you need to satisfy `GET /jobs/<id>/artifacts`.

If all three check out, the ~30% replacement + ~5% deletion holds.

## 7. Open questions

- **Artifact strategy: pull-through or mirror?** Decision blocks tracker
  rebuild scope.
- **ID-translation layer scope** — does any external system (emails,
  Slack notifications, saved searches in the UI) reference the old
  `job_id`? Each one needs translation.
- **Dashboard mirror — sensor cadence.** If we mirror Dagster asset
  metadata into the tracker DB, how stale is acceptable? Real-time
  (sensor on every materialization) vs batch (every N minutes)?