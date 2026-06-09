# Mapping current primitives → Dagster primitives

## Per-concept mapping

| Current | Dagster equivalent |
|---|---|
| `main.py` entrypoint | An `@op` that calls the same logic, OR a `k8s_job_op` wrapping the existing image |
| `EVENT_DATA` env var | Dagster run-config (`ops.<op_name>.config`) or op input |
| `publish_notification(..., PLATFORM_EVENTS_TOPIC, ...)` | Still published for external subscribers; in Dagster emitted via a thin resource so the op doesn't rebuild the client |
| `submit_kubernetes_job(...)` | Either the Dagster job itself (one run spans all stages) or `k8s_job_op` for a specific stage |
| `AdminStorageService` | A Dagster resource (`StorageResource`) injected once per run |
| `pubsub_v1.PublisherClient()` | A Dagster resource (`PubSubResource`) — one client for the whole run |
| `CodeGraphBuilder` | A Dagster resource with per-company credentials, built lazily per run |
| `langsmith_tracing(...)` | Wrap inside the op, or move to a Dagster hook that tags every run |
| `@archie_exponential_retry` | Retire in favour of Dagster's `RetryPolicy` on the op |
| `setup_maintenance_signal_handlers` | Dagster `run_monitor` + graceful shutdown hooks |
| Pub/Sub trigger from external service | Dagster `@sensor` that polls a subscription and launches a run |
| `BillingReport` GCS upload | Op returns the report; attach it as `MetadataValue.json` on the output |
| Per-job GitHub Actions deploy | One `dagster deploy` GitHub Action (or Dagster+ CI) per Definitions module |

## Example: wrapping code-downloader two ways

### Option A — minimal: `k8s_job_op` wrapping the existing image

```python
from dagster_k8s import k8s_job_op

download_code = k8s_job_op.configured({
    "image": "us-east1-docker.pkg.dev/blitzy-os-dev/gcf-artifacts/archie-job-code-downloader:latest",
    "env_vars": [
        "EVENT_DATA",
        "PROJECT_ID",
        "GCS_BUCKET_NAME",
        "PRIVATE_BLOB_NAME",
        "PLATFORM_EVENTS_TOPIC",
        "GRAPH_CODE_TOPIC",
        "GITHUB_SECRET_SERVER",
        "SERVICE_URL_ADMIN",
        # ... full list in env-contract.md
    ],
    "container_config": {
        "resources": {"requests": {"cpu": "1", "memory": "16Gi"}}
    },
}, name="download_code")
```

Pros: zero change to the job. Cons: payload still travels via env var; no
typed output in Dagster.

### Option B — native op invoking the Python entrypoint

```python
from dagster import op, Out, MetadataValue
from archie_job_code_downloader.main import process_event

@op(out={"download_result": Out()})
def download_code(context, event_data: dict, storage: StorageResource,
                  pubsub: PubSubResource, graph: GraphBuilderResource):
    result = process_event(event_data, storage, pubsub, graph)
    context.add_output_metadata({
        "files_onboarded":  MetadataValue.int(result["files_onboarded"]),
        "lines_onboarded":  MetadataValue.int(result["lines_onboarded"]),
        "file_extensions":  MetadataValue.json(result["file_extensions"]),
    })
    return result
```

Pros: typed outputs, metadata on run page, resources shared across ops.
Cons: requires refactoring each job to take resources as args (module-level
env reads go away).

## Per-job concrete mapping

| Job | Recommended op | Notes |
|---|---|---|
| code-downloader | `k8s_job_op` (slice 1) → native op (slice 2) | Long-running; k8s sizing matters |
| code-graph-generator | `k8s_job_op`; batch fan-out via dynamic outputs | Batch completion was a bespoke Neo4j query — Dagster handles fan-in natively |
| document-generator | `k8s_job_op` | Independent branch; tech-spec becomes an asset |
| reverse-file-mapper | `k8s_job_op` | Produces repo mapping + schemas assets |
| reverse-thinking-generator | `k8s_job_op`; OS fan-out with dynamic output | The two internal submissions (Linux / Windows) become branches in the DAG |
| reverse-code-generator | native op (most metrics-rich) | Attach `BillingReport` as metadata; showcase slide |
| code-generator | `k8s_job_op` | Separate job, separate schedule |

## Resource definitions to create

- `PubSubResource(project_id, topics: dict[str, str])` — centralised publisher
- `StorageResource(bucket_name, blob_name)` — wraps `AdminStorageService`
- `GraphBuilderResource(company_id_lookup: Callable)` — per-company Neo4j
- `LangSmithResource(project, endpoint, api_key)` — tracing + post-run token pulls
- `LLMResource` with model IDs — switch per env (e.g. cheap models in dev)

## Assets to promote

- `tech_spec[project_id, tech_spec_id]`
- `code_graph[company_id, repo_id, branch_id, commit_hash]`
- `repo_mapping[code_gen_id]`
- `dependency_map[code_gen_id]`
- `sorted_files[code_gen_id]`
- `reverse_code_pr[code_gen_id]` (with billing metadata)

Freshness policies: e.g. `tech_spec` must be no older than the latest
tech-spec edit; `code_graph` must be no older than the latest commit.
