# Initial prompts — Phase 1 bootstrap

> Paste these into a fresh Claude Code session, in order. Each prompt
> is self-contained — the agent does not need memory of prior ones.
> All prompts assume the session is started from inside the new
> `blitzy-orchestration/` repo (create the directory first), and that
> `claudeContext/dagster_blitzy_job/` is available as a reference
> folder (copy or symlink into the repo as `docs/context/` for the
> agent to read).

---

## Prompt 0 — Orient the agent (run once at session start)

```
Read docs/context/CONTEXT.md, docs/context/phase-plan.md, and
docs/context/local-dev-plan.md. Then summarise, in no more than
10 bullets, what Phase 1 delivers and what the local replay loop
looks like. Do not write any code yet — I want to confirm you have
the right mental model before you start.
```

Use this to sanity-check the agent before trusting it to write code.

---

## Prompt 1 — Bootstrap the orchestration repo

```
Bootstrap a new Python 3.12 Dagster project in the current directory.
Use `uv` for dependency management.

Requirements:
- Top-level package name: blitzy_orchestration
- Dependencies: dagster, dagster-webserver, dagster-k8s, dagster-gcp,
  google-cloud-pubsub, google-cloud-storage, pydantic>=2, python-dotenv
- Dev-only dependencies: pytest, pytest-asyncio, ruff
- Create a `Definitions` object exported from
  blitzy_orchestration/definitions.py. It should currently expose
  an empty `jobs=[]` list.
- Add a `.env.example` documenting every env var listed in
  docs/context/env-contract.md. Do NOT create `.env`.
- Add a `Makefile` with targets: install, fmt, lint, test, dev.
- Add a minimal `pyproject.toml` with ruff config.
- Add `.gitignore` covering .env, .venv, __pycache__, .dagster_home.

Do not touch any files outside this directory. Do not wrap the
existing archie-job-* code — that comes in the next prompt.
When done, show me the file tree and a sample `make dev` output.
```

---

## Prompt 2 — Capture EVENT_DATA from live runs

```
Build a script at scripts/capture_event_data.py that captures the
EVENT_DATA JSON payload from a live blitzy job execution and writes
it to tests/fixtures/events/<job>/<scenario>.json.

Primary strategy: query Cloud Logging for the `logger.info(...)` line
each job emits on entry (examples in docs/context/job-inventory.md).
Fallback strategy: query the Cloud Run Job execution directly via
`gcloud beta run jobs executions describe` and pull the env var.

CLI:
  python scripts/capture_event_data.py \
    --job code-downloader \
    --execution <execution-id> \
    --scenario small-public-repo \
    [--project blitzy-os-dev] [--region us-east1]

Behaviour:
- Validate that the payload parses as JSON before writing.
- Also write a sibling `<scenario>.expected.json` pre-populated with
  the top-level keys we expect to assert on
  (files_onboarded, lines_onboarded, file_extensions, batch_indexes
  for code-downloader — derive from job-inventory.md for others).
  Leave the values as the string "TODO — fill from actual run".
- Refuse to overwrite an existing fixture unless --force is passed.
- Keep the script dependency-free beyond the standard library + the
  `google-cloud-logging` SDK.

Add a pytest in tests/test_capture.py that stubs the logging client
and asserts the JSON extraction is correct across three fixtures
(normal, whitespace-wrapped, multi-line).
```

---

## Prompt 3 — Wrap `archie-job-code-downloader` as a k8s_job_op

```
Create blitzy_orchestration/jobs/code_downloader.py that wraps
archie-job-code-downloader as a Dagster `k8s_job_op` with EVENT_DATA
injected from a captured fixture file.

Requirements:
1. Define an op `download_code` using `dagster_k8s.k8s_job_op`.
   - Image tag is pulled from an env var IMAGE_CODE_DOWNLOADER
     (defaulting to the :latest tag in Artifact Registry — see
     docs/context/env-contract.md for the full path).
   - Resources: cpu 1, memory 16Gi (match current Cloud Run sizing).
   - Env var list: exactly the set from env-contract.md for this job.

2. Config schema on the op:
   - `event_data_file: str` — path to a fixture JSON under
     tests/fixtures/events/code-downloader/
   - `capture_pubsub_locally: bool = True` — when true, the op sets
     `PUBSUB_MODE=local_capture` so our forthcoming PubSubResource
     writes events to tests/output/<run_id>/events.jsonl instead of
     publishing.

3. The op should:
   a. Read the fixture, assert it parses as JSON.
   b. Inject the raw JSON string as EVENT_DATA into the Pod env.
   c. On success, parse the `DONE` event out of
      tests/output/<run_id>/events.jsonl and attach every key as
      `MetadataValue` via `context.add_output_metadata`
      (files_onboarded, lines_onboarded, file_extensions,
      batch_indexes, total_batches).

4. Define a job `archie_download_only` containing just the op.
   Export it from blitzy_orchestration/definitions.py.

5. Add a sample config file at
   config/replay/code-downloader-small.yaml pointing at a fixture
   that does not yet exist (the capture script produces it).

6. Write a README section at docs/running-locally.md explaining:
   `dagster job execute -j archie_download_only -c config/replay/...`

Do NOT modify the archie-job-code-downloader repo or its Dockerfile.
Do NOT publish to any real Pub/Sub topic by default.
```

---

## Prompt 4 — Local resource substitutes

```
Create blitzy_orchestration/resources/ with these three resources and
per-environment selectors:

1. PubSubResource (resources/pubsub.py)
   - Real mode: wraps google.cloud.pubsub_v1.PublisherClient.
   - Local mode: no-op — writes each publish() call as one JSON line
     to tests/output/<run_id>/events.jsonl.
   - Mode is selected by env var PUBSUB_MODE (real | local_capture).

2. StorageResource (resources/storage.py)
   - Wraps AdminStorageService from blitzy_platform_shared OR
     a thin google.cloud.storage client if the shared package is not
     installed locally. Detect at import time and degrade gracefully.
   - Reads are always real (dev bucket).
   - Writes go to the real bucket UNLESS STORAGE_WRITE_MODE=shadow,
     in which case they are mirrored to tests/output/<run_id>/gcs/.

3. LangSmithResource (resources/langsmith.py)
   - Disabled by default. Enabled when LOCAL_ENABLE_LANGSMITH=true.
   - When enabled, exposes the `langsmith_tracing` async context
     manager so ops can wrap LangGraph calls unchanged.

Wire all three into Definitions(resources={...}) in
blitzy_orchestration/definitions.py. Use a DAGSTER_DEPLOYMENT env var
(values: local | dev | qa | stage) to pick the resource bundle.
For anything but `local`, fall back to real resources.

Add pytest coverage in tests/resources/ for the local modes only
(no real network calls in CI).
```

---

## Prompt 5 — Verification harness

```
Build a pytest-based replay harness at tests/replay/.

Requirements:
- tests/replay/test_code_downloader.py that:
  1. Loads each fixture in tests/fixtures/events/code-downloader/
  2. Executes `archie_download_only` in-process via
     `dagster._core.test_utils.execute_in_process` or the public
     equivalent.
  3. Reads tests/output/<run_id>/events.jsonl, finds the DONE event,
     and compares the relevant metadata keys against the sibling
     `.expected.json`.
  4. Asserts:
     - run status == SUCCESS
     - lines_onboarded, files_onboarded, file_extensions,
       batch_indexes match the expected values exactly.
- A conftest.py that:
  - Sets DAGSTER_DEPLOYMENT=local, PUBSUB_MODE=local_capture,
    STORAGE_WRITE_MODE=shadow for the whole session.
  - Creates a fresh tests/output/<run_id> per test.

Add a Makefile target `make replay` that runs the harness.
Document in docs/running-locally.md the two-command loop:
  make capture JOB=code-downloader EXECUTION=<id> SCENARIO=...
  make replay
```

---

## Prompt 6 — Phase 1 exit review

```
Produce docs/phase1-exit-review.md summarising the Phase 1 status:

Include:
- Fixtures captured (list the JSON files under tests/fixtures/events/)
- Replay results: pass/fail per fixture, actual vs expected metrics
- Wall-clock comparison: prod run duration vs Dagster replay duration
  (read both from the fixture directory and the Dagster event log)
- Any deviations from the Phase 1 exit criteria in
  docs/context/phase-plan.md
- An explicit recommendation: proceed to Phase 2, iterate on Phase 1,
  or revisit the approach. Back each recommendation with evidence
  from the replay results.

Do not modify any code. This prompt only produces the review doc.
```

---

## Prompting guidelines

- **One prompt per session** when possible. They're designed to be
  atomic commits; splitting them mid-way leaves the repo inconsistent.
- **Read-first prompts** (Prompt 0, 6) should be run on a fresh
  session so the agent doesn't bring stale code assumptions.
- **Never combine** the "bootstrap" prompt with a wrapping prompt —
  the agent tends to skip the empty-Definitions step.
- **Review before running** each of Prompts 3–5: they're the ones
  that touch production-adjacent resources. Every agent-produced file
  that might publish to a real topic needs a human eyeball.
- After Prompt 5, the repo should be self-sufficient: you can loop
  capture → replay → verify without further prompting.
