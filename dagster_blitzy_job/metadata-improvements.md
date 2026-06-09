# Metadata improvements — today vs Airflow 3 / Dagster

> Consolidates the metadata audit of `archie-job-tracker` + UI with the
> native metadata surfaces of Airflow 3 and Dagster. Intended for two
> audiences: stakeholders (visibility gains) and engineering (concrete
> columns + migration path).

## 1. What the tracker captures today

### Real DB columns (indexed, queryable)

`execution_id`, `job_id`, `job_name`, `job_phase`, `job_status`,
`job_type`, `env_type`, `user_id`, `project_id`, `tech_spec_id`,
`code_gen_id`, `trigger_topic`, `is_triggered`, `exit_code`,
`exit_reason`, `time_metrics` (JSONB), plus timestamps.

### Hidden in `event_data` JSONB (stored, filterable only via text search)

`company_id`, `team_id`, `batch_index`, `branch_id`,
`git_project_repo_id`, `repo_id`, and every business metric:
`lines_onboarded`, `files_onboarded`, `files_touched`,
`lines_added / edited / removed`, `hours_saved`, `percent_complete`,
`pr_data`, `file_extensions`, `total_files_processed`, LangSmith
token counts, `BillingReport` reference.

### Derived / computed

- `job_phase` extracted from `execution_id` prefix (`job_helpers.py`)
- Time remaining scraped from Cloud Logging regex
  (`estimated_time_task.py:14-107`)
- WoW engineer counts (`engineer_metrics_service.py`)

### Aggregations

SUMs on `AggregateMetering` (entity_type=PROJECT) — just
`lines_onboarded`, `lines_generated`, `hours_saved`. No time-series,
no token cost rollup, no retry analysis.

### UI consequences

- The dashboard table shows 11 columns (email, project, execution_id,
  created, duration, ETA, phase, status, end time, actions)
- Sales Dashboard shows per-company LOC + hours saved only
- Filters (company, batch_index, user_type) hit JSONB text search —
  slow and fragile (`JobRetriggerForm.tsx:136-148`,
  `cloud_run_job_tracker_service.py:287-334`)
- No trend charts, no cost-per-PR, no token-per-run, no retry analysis
- UI shows "N/A" or "-" where JSONB fields are missing
  (`Dashboard.tsx:1082-1086`)

---

## 2. Airflow 3 native metadata

- **`DagRun` table**: `run_id`, `state`, `logical_date`,
  `data_interval_{start,end}`, `start_date`, `end_date`,
  `external_trigger`, `conf` (JSON run config), `queued_at`,
  `creating_job_id`, `dag_version_id`, `run_type`
- **`TaskInstance`**: `run_id`, `task_id`, `state`, `try_number`,
  `max_tries`, `duration`, `hostname`, `pool`, `queue`,
  `priority_weight`, `operator`, `executor_config` (JSON), `pid`,
  `rendered_map_index`, `updated_at`
- **XCom**: typed value passing between tasks (small objects)
- **Asset events** (new in 3.0): `asset_id`, `uri`, `timestamp`,
  `extra` (JSON)
- **Logs**: structured per-task
- **Plugins**: add custom columns or UI views without forking

Exposed via REST `/api/v1/dags/{id}/dagRuns` and `/taskInstances`.
No native business-metadata layer — custom fields go into `conf` or
XCom (still untyped from the UI perspective).

---

## 3. Dagster native metadata

- **`RunRecord`**: `run_id`, `status`, `run_config`, **tags** (indexed
  key-value), timestamps, pipeline_name
- **Event log** with typed `DagsterEvent` entries — step lifecycle,
  retries, failures
- **`MetadataValue`** typed fields attached to op outputs and asset
  materializations: `int`, `float`, `bool`, `url`, `markdown`,
  `path`, `json`, `table`, `timestamp`, `notebook` — rendered by
  the UI with proper formatting
- **Asset materializations**: `asset_key`,
  `materialization_metadata` (dict of MetadataValue), `partition`,
  `tags`
- **`@failure_hook` / `@success_hook`**: structured cross-cutting
  metadata attachment (the AI triage pattern)
- **GraphQL API** — every field is queryable and subscribable

Exposed via GraphQL; every metadata field is typed and the UI
renders them automatically.

---

## 4. Side-by-side gap table

| Need | Tracker today | Airflow 3 | Dagster |
|---|---|---|---|
| Run-level status + duration | ✅ columns | ✅ native | ✅ native |
| Per-step timing | ❌ (phase enum only) | ✅ TaskInstance | ✅ op event log |
| Retry count | ⚠ buried in event_data | ✅ `try_number` | ✅ RetryPolicy + event log |
| Tokens / %LOC / $ cost | ⚠ in event_data JSONB | ⚠ XCom/conf (untyped) | ✅ `MetadataValue.int/float` |
| Parent-child run correlation | ❌ none | ⚠ `external_trigger` string | ✅ tags + asset lineage |
| Error classification | ❌ none | ⚠ log string | ✅ hooks → run tags |
| Company / batch filters | ⚠ JSONB scan | ✅ conf JSON query | ✅ run tags (indexed) |
| Asset lineage (tech-spec → PR) | ❌ none | ⚠ assets (new, UI maturing) | ✅ native asset graph |
| Trend charts | ❌ none | ⚠ Grafana add-on | ✅ UI + GraphQL |
| Typed metadata in UI | ❌ flat strings | ⚠ partial | ✅ first-class |
| User-friendly value formatting | ❌ raw strings | ❌ raw strings | ✅ currency, URL, duration, markdown |

---

## 5. Quick wins — work today, no orchestrator needed

Ship before the Dagster migration; all of this also lands cleanly in
Dagster later (as `MetadataValue` / run tags).

### 5a. Promote 6 fields to indexed columns

```sql
ALTER TABLE cloud_run_job_tracker
  ADD COLUMN company_id UUID,
  ADD COLUMN team_id UUID,
  ADD COLUMN batch_index INTEGER,
  ADD COLUMN parent_execution_id TEXT,
  ADD COLUMN retry_count INTEGER DEFAULT 0,
  ADD COLUMN error_class TEXT;

CREATE INDEX idx_crjt_company_id ON cloud_run_job_tracker(company_id);
CREATE INDEX idx_crjt_team_id ON cloud_run_job_tracker(team_id);
CREATE INDEX idx_crjt_batch_index ON cloud_run_job_tracker(batch_index)
  WHERE batch_index IS NOT NULL;
CREATE INDEX idx_crjt_parent ON cloud_run_job_tracker(parent_execution_id)
  WHERE parent_execution_id IS NOT NULL;
CREATE INDEX idx_crjt_error_class ON cloud_run_job_tracker(error_class)
  WHERE error_class IS NOT NULL;
```

Backfill from existing `event_data` blob via a one-time script.
Eliminates JSONB text scans; enables true company / batch dashboards.

### 5b. Token + cost columns

```sql
ALTER TABLE cloud_run_job_tracker
  ADD COLUMN tokens_input BIGINT,
  ADD COLUMN tokens_output BIGINT,
  ADD COLUMN cost_usd NUMERIC(10, 4),
  ADD COLUMN model_name TEXT;
```

Populate via a LangSmith post-processor that runs on `DONE` events.
Unlocks the "cost per PR" and "tokens per company per day" dashboards
Sales have been asking for.

### 5c. Typed event schema (Pydantic)

Replace `event_data.get(...)` drift with a shared Pydantic model in
`blitzy-utils`:

```python
class JobEventData(BaseModel):
    project_id: str
    job_id: str
    user_id: str
    company_id: UUID
    team_id: UUID | None = None
    tech_spec_id: str | None = None
    code_gen_id: str | None = None
    batch_index: int | None = None
    parent_execution_id: str | None = None
    metadata: JobEventMetadata          # also Pydantic

class JobEventMetadata(BaseModel):
    lines_onboarded: int | None = None
    files_onboarded: int | None = None
    files_touched: int | None = None
    lines_added: int | None = None
    lines_edited: int | None = None
    lines_removed: int | None = None
    hours_saved: float | None = None
    percent_complete: float | None = None
    tokens_input: int | None = None
    tokens_output: int | None = None
    cost_usd: float | None = None
    model_name: str | None = None
    pr_data: dict | None = None
    file_extensions: dict[str, int] | None = None
```

Fails fast on unknown fields in non-prod; logs warnings in prod.
All seven `main.py` files import this once; tracker ingestion
validates against it.

### 5d. Error classifier

Add `error_class` extraction at ingestion — either a simple regex
map (`"Neo4j connection" → NEO4J_TIMEOUT`, `"rate_limit_exceeded" →
LLM_RATE_LIMIT`) or the Claude Haiku triage hook from
`ai-integration-options.md`. Groups N occurrences of the same root
cause as one incident in the UI.

### 5e. Time-series rollup materialized view

```sql
CREATE MATERIALIZED VIEW job_rollup_daily AS
SELECT
  date_trunc('day', created_at) AS day,
  job_type,
  env_type,
  company_id,
  count(*) AS total_runs,
  count(*) FILTER (WHERE job_status = 'SUCCESS') AS successes,
  count(*) FILTER (WHERE job_status = 'FAILED') AS failures,
  avg(EXTRACT(EPOCH FROM (updated_at - created_at))) AS avg_duration_s,
  percentile_cont(0.95) WITHIN GROUP
    (ORDER BY EXTRACT(EPOCH FROM (updated_at - created_at))) AS p95_duration_s,
  sum(cost_usd) AS total_cost_usd,
  sum(tokens_input) AS total_tokens_input,
  sum(tokens_output) AS total_tokens_output,
  sum(retry_count) AS total_retries
FROM cloud_run_job_tracker
GROUP BY 1, 2, 3, 4;

CREATE UNIQUE INDEX ON job_rollup_daily(day, job_type, env_type, company_id);
```

Refresh hourly via APScheduler. Replaces the hot-path aggregations
currently recomputed per request and enables trend charts in the UI.

---

## 6. What becomes free under Dagster (after migration)

Every item in section 5 maps to a native Dagster primitive:

| Quick win | Dagster equivalent |
|---|---|
| Promote columns to indexed fields | **Run tags** — key-value, indexed, filterable in UI without backend change |
| Token + cost columns | **`MetadataValue.int()` / `MetadataValue.float()`** on op output — rendered with currency/number formatting |
| Typed event schema | **Op config** via Pydantic — enforced at run launch, visible in UI |
| Error classifier | **`@failure_hook`** → `run_tags = {"ai.error_class": ...}` |
| Time-series rollup | **GraphQL API** against the event log — compute trends client-side; no materialized view |
| Asset lineage (tech-spec → code-graph → PR) | **Asset graph** — native UI, no custom dashboard |
| Trend charts | **UI + GraphQL** — every metadata field is queryable |

**Implication:** the quick-wins investment in section 5 is **not
throwaway**. Each change is something Dagster would want you to do
anyway, in a slightly different form. Start the columns now; when
the Dagster migration lands, reshape them into tags + MetadataValue.

---

## 7. Phased implementation

| Sub-phase | Work | Value | Effort |
|---|---|---|---|
| **M1** | SQL columns + indexes (5a) | Fast dashboards; eliminates JSONB scans | 1 week |
| **M2** | Backfill script + Pydantic schema (5c) | Drift fails fast; new fields auto-validated | 1 week |
| **M3** | Token + cost columns + LangSmith extractor (5b) | Cost-per-PR dashboard; per-company cost rollup | 1 week |
| **M4** | Error classifier (5d) | Grouped incidents in Slack + UI | 3 days |
| **M5** | Materialized view + trend charts in UI (5e) | Week-over-week visibility | 1 week |

Total: **~4.5 weeks**, parallelizable with the Dagster PoC (Phase 1
of the main plan). Engineering and orchestration tracks can ship
independently.

---

## 8. UI impact preview

After M1–M5:

- Dashboard table gains columns: `Cost`, `Tokens`, `Retries`, `Error class`, `Parent run`
- Filters gain: company (by UUID, not blob scan), error class, retry
  range, cost range
- Sales Dashboard gains: `Cost per PR`, `Tokens per company`, `Weekly
  cost trend` (chart)
- Engineer Dashboard gains: `Retry rate`, `Error class distribution`,
  `Duration p95` trend
- Incident view (new): grouped failures by `error_class` with
  representative traces

None of the above requires the React app to spelunk `event_data`
anymore. Every value comes from an indexed column via REST.

---

## 9. Out of scope (for now)

- Migrating historical `event_data` to a columnar store (keep as
  archive; new writes go to columns + JSONB for forward-compat)
- Replacing the Flask API surface — the additions are purely additive
- Custom UI component library changes — existing Radix/Tailwind
  components render new columns as-is
