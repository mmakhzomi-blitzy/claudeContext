# AI integration for monitoring, notification, auto-PR, and retrigger

> Companion to the Dagster migration plan. Describes how to plug LLM-based
> automation into Dagster (or Airflow) once the orchestrator owns the
> events. Written as a reference — pick the slices that match the current
> priorities.

## Why this matters now

Once the orchestrator owns execution events, we have **structured signals**
for the first time: typed run status, op-level metadata, retry counts,
cost, duration, and tags. That's exactly the shape LLMs consume well.
Retrofitting the same automation onto the current Pub/Sub-based fleet
would require building the event model first.

Blitzy specifically has two shortcuts worth noting:

- Anthropic, OpenAI, and Google API keys are already in every job's env.
- `archie-job-reverse-code-generator` **already generates PRs** — the
  infrastructure we're considering adopting is literally the company's
  product. Pointing it at internal infra repos is a direct dogfood.

## The four areas, at a glance

| Area | What "lightweight" looks like | Keystone dependency |
|---|---|---|
| Monitoring | Per-failure triage LLM call → structured metadata + tags | — |
| Notification | LLM-rendered Slack messages grouped by error class | Triage output |
| Auto-PR | Claude Code Action opens a draft PR from the failure + logs | Triage output + narrow safe-list |
| Retrigger | Classifier gates automatic retry; guardrails prevent runaways | Triage output + budget tags |

**Everything builds on the triage hook.** Build it first; the other three
are thin consumers of its output.

---

## Area 1 — Monitoring

### Goal

Classify every failure into a small, stable vocabulary so downstream
tools can act on it. Humans also benefit: "Neo4j timeout" is more useful
than "subprocess returned 137."

### Options, in order of lift

1. **Triage LLM per failed run (recommended starting point).**
   A Dagster `@failure_hook` or Airflow `on_failure_callback` reads the
   exception + tail of logs, calls Claude Haiku, and writes structured
   metadata. Cost: ~$0.001 per failure.

2. **Scheduled anomaly rollup.**
   A daily job queries the last 7 days of runs (duration, retry count,
   token cost, success rate) and asks the LLM to describe deviations.
   Output posts to Slack weekly.

3. **SaaS AIOps (Datadog AI, New Relic AI, PagerDuty AIOps).**
   Fast to demo, harder to customise to Blitzy's token-cost and %LOC
   signals. Consider only after (1) and (2) prove signal.

### Triage hook — reference implementation

```python
# blitzy_orchestration/hooks/triage.py
import json
from dagster import failure_hook, HookContext
from anthropic import Anthropic

_SYSTEM = """You triage CI/CD job failures. Read the log tail and output
JSON only. Schema:
{
  "error_class": one of [
    "NEO4J_TIMEOUT", "LLM_RATE_LIMIT", "LLM_CONTEXT_OVERFLOW",
    "GCS_PERMISSION", "NETWORK", "OOM", "TIMEOUT",
    "DEPENDENCY_VERSION", "MISSING_SECRET", "UNKNOWN"
  ],
  "is_transient": true | false,
  "suspected_file": str | null,
  "one_liner": str,       // <= 140 chars
  "confidence": float     // 0.0 – 1.0
}
"""

@failure_hook(required_resource_keys={"anthropic"})
def triage_failure(context: HookContext) -> None:
    client: Anthropic = context.resources.anthropic
    logs = context.instance.all_logs(
        context.run_id, of_type={"ERROR", "CRITICAL"}
    )
    tail = "\n".join(e.message for e in logs[-200:])

    try:
        resp = client.messages.create(
            model="claude-haiku-4-5-20251001",
            max_tokens=400,
            system=_SYSTEM,
            messages=[{"role": "user", "content": tail}],
        )
        triage = json.loads(resp.content[0].text)
    except Exception as e:
        context.log.warning(f"triage failed: {e}")
        return

    context.instance.add_run_tags(context.run_id, {
        "ai.error_class":   triage["error_class"],
        "ai.is_transient":  str(triage["is_transient"]),
        "ai.confidence":    f"{triage['confidence']:.2f}",
        "ai.one_liner":     triage["one_liner"][:140],
    })
    context.log.info("ai_triage", extra={"triage": triage})
```

Attach once at the `Definitions` level and every op inherits it.

### What to measure

- Coverage: fraction of failures that got a triage
- Precision: human-labelled accuracy of `error_class` on a weekly sample
- Cost: $ per failure (should be <$0.005)
- Staleness: time from run failure → tag write (should be <30s)

---

## Area 2 — Notification

### Goal

Replace the current "one Slack message per FAILED event" pattern with
grouped, context-rich alerts that make oncall's job easier, not noisier.

### Design

A Dagster sensor (or Airflow DAG) wakes every 5 minutes, pulls recent
failed runs that haven't been summarised yet, groups by
`ai.error_class`, and posts a single consolidated Slack message with:

- Per-class count
- Representative `ai.one_liner`
- Links to the first run and the UI filter for all of them
- The shortest reproducer (top 3 lines of the stack trace)
- Confidence marker and **"AI-generated, verify before acting"** label

### Reference sensor

```python
from collections import defaultdict
from dagster import sensor, RunFailureSensorContext, SkipReason

@sensor(minimum_interval_seconds=300)
def grouped_failure_notifier(context):
    recent = context.instance.get_runs(
        filters=RunsFilter(
            statuses=[DagsterRunStatus.FAILURE],
            updated_after=context.last_tick_completion_time,
        ),
        limit=50,
    )
    if not recent:
        return SkipReason("no new failures")

    grouped = defaultdict(list)
    for run in recent:
        grouped[run.tags.get("ai.error_class", "UNKNOWN")].append(run)

    lines = []
    for cls, runs in sorted(grouped.items(), key=lambda x: -len(x[1])):
        sample = runs[0]
        lines.append(
            f"• *{cls}* × {len(runs)} — "
            f"<{dagster_url(sample.run_id)}|{sample.run_id[:8]}> "
            f"{sample.tags.get('ai.one_liner', '')}"
        )

    slack_post(
        channel="#blitzy-oncall",
        text="\n".join(lines) + "\n_AI-generated — verify before acting._",
    )
```

### What to avoid

- **Do not** post the raw LLM output without grouping — it defeats the
  purpose. The win is summarisation, not elaboration.
- **Do not** treat `ai.error_class == UNKNOWN` as a group — surface
  those individually. Pattern blindness is worse than pattern confusion.
- **Do not** auto-page on AI output alone. Use it to *prioritise*
  existing alerts, not replace them.

---

## Area 3 — Auto-PR for fixing issues

This is the area with the **highest blast radius**. Conservative defaults
matter more than fancy automation.

### Phased safe-list (expand one class at a time)

1. **Start here:** dependency version bumps suggested by Dependabot that
   broke the build — re-pin to the last known good.
2. Timeout or memory bumps for jobs that fail with OOM / timeout and
   have no other symptom.
3. Env-var typos — a `KeyError: 'GCS_BUKCET_NAME'` triggers a PR against
   the deployment yaml.
4. Missing Secret Manager references when a job logs a secret-lookup
   failure.
5. Flaky-test quarantine: mark a test `skip` with a `# TODO(AI)` note.

### Never automate

- Logic bugs inside a job
- Data-model or schema changes
- Anything touching billing / metering code paths
- Breaking-change dependency upgrades
- Cross-repo refactors
- Anything in `main` infra without human review

### Flow

```
Dagster @failure_hook
  → match on ai.error_class ∈ {DEPENDENCY_VERSION, TIMEOUT, MISSING_SECRET}
  → Claude Code GitHub Action (via workflow_dispatch)
      - Reads: failed run metadata, log tail, repo context
      - Produces: a diff targeting the matched safe-list class
      - Opens: **draft** PR with label "ai-generated", CODEOWNERS assigned
  → CODEOWNERS reviews → human merges
```

No auto-merge, ever. Even for the narrowest classes.

### Dogfood path

For infra repos (the seven `archie-job-*` repos), consider routing
through `archie-job-reverse-code-generator` itself with a tech-spec
that describes the fix. Same product, new use case. Treat it as a
validation of the product's general-purpose capabilities.

---

## Area 4 — Smart retrigger

### Goal

Auto-recover from transient failures without burning LLM budget or
masking real problems.

### Decision policy

Auto-retry **if and only if**:

- `ai.is_transient == true`
- `ai.confidence >= 0.7`
- `retry_count < 3`
- `cumulative_cost_usd < budget_cap` (per project)
- Run is **not** tagged `expensive=true` (e.g. full reverse-code
  generation for a large repo)
- Per-project rate limit not exceeded (e.g. max 10 auto-retries / hour)

Everything else goes to the existing human-in-the-loop path.

### Reference sensor

```python
@sensor(job=archie_full_build, minimum_interval_seconds=60)
def auto_retrigger(context):
    runs = context.instance.get_runs(
        filters=RunsFilter(statuses=[DagsterRunStatus.FAILURE]),
        limit=20,
    )
    for run in runs:
        if context.cursor_contains(run.run_id):
            continue
        t = run.tags
        if not (
            t.get("ai.is_transient") == "True"
            and float(t.get("ai.confidence", "0")) >= 0.7
            and int(t.get("retry_count", "0")) < 3
            and float(t.get("cumulative_cost_usd", "0")) < 5.00
            and "expensive" not in t
        ):
            context.log.info(f"skipping {run.run_id} — guardrail failed")
            continue

        yield RunRequest(
            run_key=f"auto-retry-{run.run_id}",
            run_config=run.run_config,
            tags={
                **t,
                "retry_count": str(int(t.get("retry_count", "0")) + 1),
                "retry_parent": run.run_id,
                "retry_reason": t.get("ai.error_class", "UNKNOWN"),
            },
        )
        context.update_cursor_append(run.run_id)
```

### Measurement

- % of retries that succeed on attempt ≥ 2
- $ spent on auto-retries (weekly)
- Ratio of auto-retries to human retriggers — should trend upward
- Runaway incidents (should be zero after the first week of tuning)

---

## Risks and guardrails (master list)

| Risk | Mitigation |
|---|---|
| Runaway auto-retry burns tokens | Hard cap per project per day; `max_cost_per_run_chain` tag |
| PR spam | ≤ N auto-PRs / repo / day; `draft` status; CODEOWNERS required; `ai-generated` label |
| Hallucinated root causes | Link every summary to raw trace; "AI-generated, verify" label; confidence threshold of 0.7 |
| Feedback loops (AI → retrigger → fail → AI…) | Hard stop at `retry_count >= 3`; weekly decision audit |
| Vendor lock-in (SaaS AIOps) | Build in-house first; only adopt SaaS once signal is proven |
| PII / secrets leaking to LLM | Scrub with regex before sending (token-like strings, credentials); prefer models with zero-data-retention |
| Alert fatigue from AI | Group by error class; never add a new AI channel — reuse the existing oncall channel |
| Over-trust in AI classification | Weekly human sampling: 20 labeled failures reviewed, precision tracked |
| Cost opacity | Every AI call tagged to a Dagster run; cost dashboard per day / per project |

---

## Phased rollout

| AI-Phase | Weeks | Scope | Exit criteria |
|---|---|---|---|
| A | 1 | Triage hook wired to Anthropic resource | ≥80% of failures get a triage tag |
| B | 1 | Grouped Slack notifications | Oncall survey: noise ↓, signal ↑ |
| C | 2 | Auto-retrigger (transient only) with guardrails | ≥30% of failures auto-resolved; zero runaway-cost incidents |
| D | 3 | Auto-PR for one safe-list class (start with DEPENDENCY_VERSION) | <10% false-positive rate on draft PRs over 20 samples |
| E | 2 | Anomaly rollup across token, %LOC, duration | At least one real regression surfaced by the rollup |

Total: ~9 weeks, mostly gated on signal quality from Phase A.
These phases are **post-Phase 7** of the Dagster migration — they
require the orchestrator to own events first.

### Why A alone unlocks most of the value

Once failures carry `ai.error_class`, `ai.is_transient`, and
`ai.one_liner`, existing tools can already:

- Route Slack messages by class (no AI integration needed for the
  router)
- Gate human retriggers on `is_transient` (saves the human one click)
- Query "how many NEO4J_TIMEOUT this week" from the Dagster GraphQL API
- Feed a weekly Looker/Grafana dashboard

The LLM-driven phases B–E are higher leverage than A, but A is the
cheapest and lowest-risk; ship it first.

---

## Integration with the current tracker

While the Dagster migration is in progress, the AI layer can also
consume the tracker's existing events:

- Triage hook can be a **separate microservice** that subscribes to
  `PLATFORM_EVENTS_TOPIC` and writes back to the `cloud_run_job_tracker`
  table via new columns (`ai_error_class`, `ai_is_transient`, …).
- The UI's existing job-list table can render the new columns as badges
  without any frontend refactor.
- Smart retrigger can use the tracker's `POST /v1/job/{id}/trigger`
  endpoint instead of Dagster's `launchRun` — same semantics.

This means **AI-Phase A is independently valuable even if Dagster
adoption is deferred.** It's the one part of this plan you could ship
against the current architecture today.

---

## Out of scope

- Generative analytics ("describe the week in data") — cool, not urgent
- LLM-based test generation for the seven jobs — separate workstream
- Voice/chat interface for the dashboard — experiment, not backbone
- Fully autonomous incident resolution — we are explicitly keeping
  humans in the loop for anything that touches `main` or costs
  meaningful tokens
