# Phase 1 — Implementation

> Companion to `PLAN.md` §6 and `phase-plan.md`. This is the concrete
> Phase 1 spec that was built under `/Users/m.makhzomi/myProject/BlitzyDagster`.
> Captures what changed from the scaffold, how the launcher abstraction works,
> and how to run the `archie-job-code-downloader` against any of the three
> execution surfaces.

**Status:** Phase 1 scaffolding — `archie-job-code-downloader` wraps cleanly
across all three surfaces; Pub/Sub behaviour is flag-controlled; EVENT_DATA
and env are manually injectable per run.

---

## Design requirements (agreed 2026-04-23)

1. **Three execution surfaces, one op body.** The same Dagster op must be
   runnable as (a) a local Python subprocess, (b) a `docker run` against the
   pre-built job image, and (c) a k8s Job — where (c) works for both Docker
   Desktop's local cluster and real GKE with no code change (just a kubeconfig
   context swap).
2. **Manual injection.** `EVENT_DATA` and any other env var must be settable
   per run without editing code — from inline JSON, a fixture file, or a
   Cloud Logging lookup.
3. **Pub/Sub left alone.** The job image is NOT modified. Its Pub/Sub
   publishes are routed by injecting `PUBSUB_EMULATOR_HOST` (or not) from
   Dagster, gated by a single `DAGSTER_PUBSUB_MODE` flag.
4. **Repo is `blitzy_orchestration`.** Scaffold package `dagster_quickstar`
   was renamed and contents moved to the repo root.

---

## Repo layout

```
BlitzyDagster/
├── pyproject.toml                # deps: dagster, dagster-k8s, dagster-gcp,
│                                 # kubernetes, google-cloud-pubsub, google-cloud-logging,
│                                 # google-cloud-storage, pydantic-settings, python-dotenv
├── Makefile                      # install / dev / pubsub-emulator / capture / replay
├── docker-compose.yaml           # pubsub-emulator service
├── .env.example                  # every env var from env-contract.md
├── .gitignore                    # + .env, tests/output, .venv
├── README.md                     # quickstart for all three surfaces
├── config/
│   ├── local.yaml                # subprocess launcher + pubsub emulator
│   ├── docker.yaml               # docker launcher + pubsub emulator
│   ├── k8s-local.yaml            # k8s launcher (Docker Desktop context)
│   └── k8s-gcp.yaml              # k8s launcher (GKE context) + pubsub real
├── scripts/
│   ├── capture_event_data.py     # Cloud Logging → fixture JSON
│   └── create_pull_secret.sh     # Docker Desktop imagePullSecret helper
├── src/blitzy_orchestration/
│   ├── __init__.py
│   ├── definitions.py            # Dagster Definitions (jobs + resources)
│   ├── settings.py               # Pydantic Settings for the job env contract
│   ├── resources/
│   │   ├── __init__.py
│   │   ├── launcher.py           # JobLauncher + 3 impls
│   │   ├── pubsub.py             # PubSubResource (emulator | real)
│   │   └── env.py                # JobEnvResource (typed env composer)
│   └── defs/
│       ├── __init__.py
│       ├── resources.py          # build_resources(deployment) — factory
│       └── code_downloader.py    # download_code op + archie_download_only job
└── tests/
    ├── __init__.py
    └── fixtures/events/code-downloader/.gitkeep
```

---

## The launcher abstraction

### Contract

```python
class LaunchSpec(BaseModel):
    name: str                          # short id, e.g. "code-downloader"
    image: str | None = None           # required for docker + k8s
    local_working_dir: str | None = None   # required for subprocess
    local_entrypoint: list[str] = ["python", "main.py"]
    env: dict[str, str]                # full env, already composed
    resources: LaunchResources         # cpu, memory (k8s only)
    timeout_seconds: int = 86400

class LaunchResult(BaseModel):
    exit_code: int
    stdout: str
    stderr: str

class JobLauncher(ConfigurableResource, ABC):
    @abstractmethod
    def launch(self, spec: LaunchSpec, context) -> LaunchResult: ...
```

### Three implementations

| Launcher | How it runs | Gotchas |
|---|---|---|
| `LocalSubprocessLauncher` | `subprocess.run(["python", "main.py"], cwd=..., env=...)` | Needs the job repo's venv with `blitzy-platform-shared` installed from Artifact Registry |
| `DockerLauncher` | `docker run --rm -e ... <image>` | Emulator host is `host.docker.internal` on Mac/Windows; `--add-host=host.docker.internal:host-gateway` on Linux |
| `K8sJobLauncher` | `BatchV1Api.create_namespaced_job(...)`, poll, tail logs | Needs kubeconfig context set; `imagePullSecrets` required for Blitzy Artifact Registry; secrets injected via `secret_env_refs` |

### Selection

Highest-priority wins:

1. `ops.download_code.config.launcher: local|docker|k8s` (per-run)
2. `DAGSTER_EXECUTION_TARGET=local|docker|k8s` (env var)
3. Default: `local` under `dagster dev`, `k8s` otherwise

The op receives the launcher via the `launcher` resource key; the factory in
`defs/resources.py:build_resources()` reads `DAGSTER_DEPLOYMENT` and
`DAGSTER_EXECUTION_TARGET` and picks the right concrete class.

---

## Pub/Sub: the flag

| Mode | What happens | When to use |
|---|---|---|
| `emulator` | `PUBSUB_EMULATOR_HOST` injected into the job env; after the run, Dagster drains the emulator subscriptions and attaches the `DONE` event fields as metadata | Local + Docker + k8s-local — default everywhere except GCP |
| `real` | No injection; job publishes to the real dev Pub/Sub topics. Downstream `archie-job-*` services keep consuming | k8s-GCP when you want the full chain to flow, or one-off prod-parity tests |

Selected by `DAGSTER_PUBSUB_MODE=emulator|real` or per-run config override
(`resources.pubsub.config.mode`).

Emulator lifecycle is external — run `make pubsub-emulator` (docker-compose)
before `dagster dev`. The `PubSubResource` only talks to it; it does not
own the process.

### Emulator host resolution per surface

| Surface | `PUBSUB_EMULATOR_HOST` injected into child |
|---|---|
| Local subprocess | `localhost:8085` |
| Docker | `host.docker.internal:8085` (Mac/Win) / gateway IP on Linux |
| k8s Docker Desktop | ClusterIP service `pubsub-emulator.default.svc.cluster.local:8085` (emulator must be deployed into the cluster — see `make pubsub-emulator-k8s`) |
| k8s GKE | N/A — use `real` mode |

---

## EVENT_DATA: three input paths

The `download_code` op config takes one of these, checked in order:

1. `event_data_inline: str` — raw JSON string in the run config.
2. `event_data_file: str` — path relative to repo root (usually
   `tests/fixtures/events/code-downloader/<scenario>.json`).
3. `event_data_from_execution: {job, execution_id}` — resolves via
   `scripts/capture_event_data.py` at op-start time.

Other env vars come from `JobEnvResource`, a `pydantic-settings` model
populated from `.env` + `DAGSTER_DEPLOYMENT`-specific overrides. Per-run
env additions can be passed via `ops.download_code.config.env_overrides`.

---

## Metadata extracted after a run

After `launch()` returns exit code 0, the op:

1. If `pubsub.mode == emulator`: drains the `PLATFORM_EVENTS_TOPIC` and
   `GRAPH_CODE_TOPIC` subscriptions, finds the `DONE` event for this
   `job_id`, parses the payload.
2. If `pubsub.mode == real`: skips the pull; metadata comes from parsing
   stdout (the downloader logs `lines_onboarded=...` near the end).
3. Attaches to the op via `context.add_output_metadata`:
   - `lines_onboarded` (int)
   - `files_onboarded` (int)
   - `total_files` (int)
   - `file_extensions` (json)
   - `batch_indexes` (json)
   - `total_batches` (int)

---

## How to run

### One-time setup

```bash
cd /Users/m.makhzomi/myProject/BlitzyDagster
uv sync                                   # install deps
cp .env.example .env                      # fill in GCP project, topics, creds
gcloud auth application-default login     # for real-mode reads from dev GCS
gcloud auth configure-docker \
    us-east1-docker.pkg.dev               # for docker launcher image pull
```

For the **k8s launcher on Docker Desktop**:

```bash
# Enable Kubernetes in Docker Desktop preferences first.
kubectl config use-context docker-desktop
./scripts/create_pull_secret.sh           # creates blitzy-ar-pull secret
```

For **k8s on GKE**:

```bash
gcloud container clusters get-credentials <cluster> --region us-east1
kubectl config use-context gke_blitzy-os-dev_us-east1_<cluster>
```

### Start the Pub/Sub emulator (all local surfaces)

```bash
make pubsub-emulator
# → starts gcr.io/google.com/cloudsdktool/cloud-sdk on :8085
```

### Start Dagster

```bash
make dev
# → dagster dev at http://localhost:3000
```

### Capture an EVENT_DATA fixture from a real dev run

```bash
make capture \
    JOB=code-downloader \
    EXECUTION=<cloud-run-execution-id> \
    SCENARIO=small-public-repo
# writes tests/fixtures/events/code-downloader/small-public-repo.json
# and a sibling .expected.json skeleton
```

### Run each surface

```bash
# Local subprocess
DAGSTER_EXECUTION_TARGET=local \
    dagster job execute -j archie_download_only -c config/local.yaml

# Docker
DAGSTER_EXECUTION_TARGET=docker \
    dagster job execute -j archie_download_only -c config/docker.yaml

# k8s (Docker Desktop context)
kubectl config use-context docker-desktop
DAGSTER_EXECUTION_TARGET=k8s \
    dagster job execute -j archie_download_only -c config/k8s-local.yaml

# k8s (GKE context) — uses real Pub/Sub so the chain flows
kubectl config use-context gke_blitzy-os-dev_us-east1_<cluster>
DAGSTER_EXECUTION_TARGET=k8s DAGSTER_PUBSUB_MODE=real \
    dagster job execute -j archie_download_only -c config/k8s-gcp.yaml
```

### Flip Pub/Sub mode without changing code

```bash
DAGSTER_PUBSUB_MODE=real dagster job execute ...   # publishes to dev topics
DAGSTER_PUBSUB_MODE=emulator dagster job execute ...  # default
```

---

## Exit criteria

Phase 1 is done when:

- [ ] `archie_download_only` runs green on all three surfaces against the
      same fixture.
- [ ] Run page shows: `lines_onboarded`, `files_onboarded`, `total_files`,
      `file_extensions`, `batch_indexes`, `total_batches`.
- [ ] `DAGSTER_PUBSUB_MODE=real` produces observable Pub/Sub messages on
      dev; `=emulator` produces none on dev and yields non-empty metadata.
- [ ] `scripts/capture_event_data.py` successfully pulls a payload from
      Cloud Logging by execution id.

---

## What Phase 1 does NOT do

- No change to any `archie-job-*` source or Dockerfile.
- No native `@op` refactor (that's Phase 5).
- No sensors, no schedules, no assets yet.
- No multi-env Definitions — single `Definitions` object; env selection is
  via resource factory picking up `DAGSTER_DEPLOYMENT`.
- No CI GitHub Action yet (next tick after Phase 1 green locally).

---

## Known gotchas

- `blitzy-platform-shared` lives in a private Artifact Registry. The
  subprocess launcher assumes the job repo's venv already has it installed
  (it does today, since that's how jobs run locally). Docker + k8s pull
  from Artifact Registry directly — `imagePullSecret` required on k8s.
- The job reads `os.environ[...]` at module-import time. We must set every
  required env var before `python main.py` starts; a missing var will
  `KeyError` during import. `JobEnvResource` enforces this with Pydantic.
- Emulator mode does NOT cover Neo4j, GitHub, or LLM side-effects — those
  still hit real dev. Treat Phase 1 as "safe Pub/Sub, real everything
  else." A later phase can shadow GCS writes.
- K8s Job timeout must match the job's Cloud Run timeout (24h). Set via
  `spec.timeout_seconds` → `activeDeadlineSeconds`.
