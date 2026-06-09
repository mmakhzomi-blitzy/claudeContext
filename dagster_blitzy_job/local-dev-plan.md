# Local dev loop — capture, replay, verify

> Companion to `phase-plan.md`. This is the day-to-day workflow for
> iterating on the Dagster migration before anything is deployed.

## The loop

```
┌─────────────────┐    ┌──────────────┐    ┌──────────────────┐    ┌───────────────┐
│  Run job in     │ →  │  Capture     │ →  │  Replay under    │ →  │  Verify       │
│  dev (Cloud     │    │  EVENT_DATA  │    │  Dagster locally │    │  metrics &    │
│  Run or k8s)    │    │  locally     │    │  (same payload)  │    │  side-effects │
└─────────────────┘    └──────────────┘    └──────────────────┘    └───────────────┘
```

Run → Capture → Replay → Verify. Each step is cheap; the whole loop
should be tight enough to run 10× a day during Phase 1.

---

## Step 0 — Prereqs (one-time)

Install / configure on the dev machine:

- Python 3.12 + `uv` or `poetry` for the `blitzy-orchestration` repo
- GCP Application Default Credentials:
  `gcloud auth application-default login`
- Service account key for `blitzy-os-dev` (reuse the `service-key-dev.json`
  used by the existing job repos), referenced as `GOOGLE_APPLICATION_CREDENTIALS`
- `gcloud` CLI with `blitzy-os-dev` as default project
- Docker (for Phase-1 `k8s_job_op` local test — either kind/minikube
  for a real Pod, or `docker run` to shortcut the image locally)
- Read access to: dev GCS buckets, dev Neo4j, dev Artifact Registry,
  dev Pub/Sub topics (publish **not** needed in replay mode)
- `.env` at the repo root holding the same keys the Cloud Run deploy
  injects (see `env-contract.md`). Never commit it.

A lightweight alternative to Docker/k8s for early Phase-1: run the
job's `main.py` directly in a sub-process against the captured
EVENT_DATA. Skips container parity but gets fastest iteration.

---

## Step 1 — Capture EVENT_DATA from a live run

Every job logs its input near the top of `main.py`:

```python
logger.info(f"Processing notification data: {EVENT_DATA}")   # code-downloader
logger.info(f"Generating document for notification data: {EVENT_DATA}")  # doc-gen
# ...
```

That gives us four capture paths, in order of preference:

### 1a. Cloud Logging (simplest, always available)

```bash
# get the log line that contains the payload, pipe to a file
gcloud logging read \
  'resource.type="cloud_run_job"
   AND resource.labels.job_name="code-downloader"
   AND labels."run.googleapis.com/execution_name"="<execution-id>"
   AND textPayload:"Processing notification data"' \
  --project blitzy-os-dev \
  --format='value(textPayload)' \
  --limit 1 \
  | sed -E 's/^[^{]*(\{.*\})[^}]*$/\1/' \
  > tests/fixtures/events/code-downloader/<scenario>.json
```

Wrap this in `scripts/capture_event_data.py` (see Initial Prompt 2).

### 1b. Cloud Run execution env inspection

```bash
gcloud beta run jobs executions describe <execution-id> \
  --region us-east1 --project blitzy-os-dev \
  --format='value(spec.template.spec.template.spec.containers[0].env)'
```

Useful when the logs have been rotated.

### 1c. Pub/Sub peek subscription

Stand up a read-only secondary subscription on `GRAPH_CODE_TOPIC` /
`GENERATE_*_TOPIC` that nacks every message after copying the body to
a local file. Lets you capture upcoming triggers without waiting for
someone to run a build. Overkill for Phase 1.

### 1d. Self-capture inside the job

Temporarily add a line to the job's entry:

```python
AdminStorageService.write_fixture("event_data", EVENT_DATA)
```

Only needed if (1a)–(1c) don't surface the right payload. Avoid for
anything with PII.

### Fixture layout

```
tests/fixtures/events/
├── code-downloader/
│   ├── small-public-repo.json
│   ├── retrigger.json
│   ├── whitelist-updated.json
│   └── mono-repo-10k-files.json
├── code-graph/
│   └── batch-0-of-4.json
├── document-generator/
│   ├── with-figma.json
│   └── plain-text-spec.json
├── reverse-file-mapper/…
├── reverse-thinking-generator/…
├── reverse-code-generator/…
└── code-generator/…
```

Pair each fixture with a sibling `.expected.json` capturing the
metrics we expect from replay (files_onboarded, lines_onboarded, …).

---

## Step 2 — Feed EVENT_DATA into Dagster

Two replay styles. Both live inside `blitzy-orchestration`.

### 2a. k8s_job_op wrapper (default for Phase 1)

The op reads a fixture path from Dagster run config, loads the JSON,
and sets it as the `EVENT_DATA` env var when launching the container.

```yaml
# config/replay/code-downloader-small.yaml
ops:
  download_code:
    config:
      event_data_file: tests/fixtures/events/code-downloader/small-public-repo.json
```

Run:

```bash
dagster job execute -m blitzy_orchestration \
  -j archie_download_only \
  -c config/replay/code-downloader-small.yaml
```

### 2b. Direct-Python replay (fastest iteration)

Skip the container; call the job's `process_event(event_data_str)`
directly from a Dagster op that imports it. Caveat: requires the job's
repo checked out or installed as a package.

```python
from archie_job_code_downloader.main import process_event

@op(config_schema={"event_data_file": str})
def download_code_native(context):
    payload = Path(context.op_config["event_data_file"]).read_text()
    process_event(payload)
```

Use for debugging before the Docker image is rebuilt. Always retest
through 2a before closing the loop.

---

## Step 3 — Local resource substitutes

Define resources twice: real (dev), no-op (local replay). `dagster dev`
picks the local set via `DAGSTER_DEPLOYMENT=local`.

| Resource | Local (replay) | Dev |
|---|---|---|
| `StorageResource` (GCS) | **Real, read** (dev bucket) — writes point to `tests/output/<run_id>/…` | Real |
| `PubSubResource` | No-op publisher: writes each event to `tests/output/<run_id>/events.jsonl` | Real |
| `GraphBuilderResource` (Neo4j) | Pick: real dev instance OR local Docker `neo4j:5` | Real |
| `LangSmithResource` | Disabled unless `LOCAL_ENABLE_LANGSMITH=true` | Real |
| `LLMResource` | Real keys (cheap models) OR recorded-cassette fake | Real |
| `GithubSecretServer` | Real dev | Real |

**Side-effect rule:** a replay must not publish to any `*_TOPIC` that
a real subscriber consumes. The no-op publisher enforces this by
default. If a fixture *needs* a real publish to exercise the next
stage, override it explicitly per-run.

---

## Step 4 — Verification checklist (Phase 1 · code-downloader)

Compare replay output against the `.expected.json` captured alongside
the fixture:

- [ ] Run completes without exceptions
- [ ] `lines_onboarded` matches within ±0
- [ ] `files_onboarded` matches within ±0
- [ ] `file_extensions` dict matches exactly
- [ ] `batch_indexes` list matches (same count & order)
- [ ] GCS fixture output folder contains one file per batch
- [ ] No real Pub/Sub publishes to prod topics (check
      `tests/output/<run_id>/events.jsonl` — expect ≥1 entries)
- [ ] Replay wall-clock is ≤ 1.5× the original run

Automate via a single Pytest:

```python
def test_code_downloader_replay(tmp_path):
    run_id = execute_dagster_job("archie_download_only",
                                 fixture="small-public-repo.json")
    metadata = read_run_metadata(run_id)
    expected = json.loads(Path("tests/fixtures/events/"
                               "code-downloader/small-public-repo"
                               ".expected.json").read_text())
    for key in ("lines_onboarded", "files_onboarded", "file_extensions"):
        assert metadata[key] == expected[key], key
```

---

## Step 5 — Iterating

Day-to-day loop:

1. `make capture JOB=code-downloader EXECUTION=<id>` — grabs the
   payload + expected metrics from a live run.
2. `dagster dev` — opens the UI.
3. Trigger the Dagster job with the fresh fixture.
4. Compare actual vs expected in the UI; iterate on `build_env_vars`
   / resource config.
5. `pytest tests/replay/` — gate before pushing.
6. When green on all captured scenarios, graduate the fixture from
   `tests/fixtures/` into the shared set that CI runs on every PR.

---

## Secrets & credentials — the rules

- All secrets stay in `.env` (git-ignored) or Secret Manager references.
- `service-key-dev.json` lives in `~/.config/blitzy/` — **never** in
  the orchestration repo.
- Real LLM API keys are loaded only when `LOCAL_ENABLE_LLMS=true`.
  Default replays use recorded cassettes.
- The no-op Pub/Sub publisher is the default. A PR that changes the
  default needs an explicit review comment.

---

## What Phase 1 does *not* need yet

- Actual k8s cluster (Docker-only or direct-Python is fine)
- Full Dagster+ setup (OSS `dagster dev` is enough for replay)
- CI that spins up Pub/Sub / Neo4j emulators (add in Phase 2)
- Production-parity VPC / egress config (Phase 6)

The loop is deliberately thin so the first week is spent on replay
correctness, not infra plumbing.
