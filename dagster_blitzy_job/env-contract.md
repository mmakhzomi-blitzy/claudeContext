# Env vars each job reads

Extracted from the `os.environ["..."]` lines in each `main.py` and the
deployment `gcloud run jobs deploy --set-env-vars` blocks.

## Shared across (nearly) all jobs

| Var | Purpose |
|---|---|
| `EVENT_DATA` | JSON payload containing project_id, repo_name, job_id, propagate, user/team/company ids, branch info, commit hashes, tech_spec_id, code_gen_id, and job-specific flags |
| `PROJECT_ID` | GCP project id |
| `GCS_BUCKET_NAME` | Primary GCS bucket |
| `BLOB_NAME` or `PRIVATE_BLOB_NAME` | Per-environment blob prefix |
| `PLATFORM_EVENTS_TOPIC` | Pub/Sub topic for status events |

## Per-topic wiring

| Var | Jobs that use it |
|---|---|
| `GRAPH_CODE_TOPIC` | downloader, code-graph-generator |
| `GENERATE_DOCUMENT_TOPIC` | document-generator |
| `GENERATE_REVERSE_THINKING_TOPIC` | reverse-file-mapper |
| `GENERATE_REVERSE_CODE_TOPIC` | reverse-code-generator |
| `GENERATE_REVERSE_DOCUMENT_TOPIC` | code-graph-generator |
| `UPLOAD_CODE_TOPIC` | code-generator |
| `GENERATE_CODE_TOPIC` | code-generator |

## External services

| Var | Jobs |
|---|---|
| `GITHUB_SECRET_SERVER` | downloader, graph, file-mapper, reverse-thinker, reverse-code |
| `NEO4J_SERVER` / `NEO4J_USERNAME` / `NEO4J_PASSWORD` | same set (company-specific overrides fetched at runtime via `get_company_neo4j_instance_credentials`) |
| `MARKDOWN_SERVER` | document-generator |
| `SERVICE_URL_ADMIN` | all (via `blitzy_utils.service_client`) |
| `SERVICE_URL_GITHUB`, `SERVICE_URL_RELAY` | downloader |

## LLM + tracing

| Var | Jobs |
|---|---|
| `ANTHROPIC_API_KEY` | document, code-graph, reverse-file-mapper, reverse-thinking, reverse-code, code-generator |
| `OPENAI_API_KEY` | same set + downloader |
| `GOOGLE_API_KEY` | document, reverse-*, code-graph, code-generator |
| `VOYAGE_API_KEY` | code-graph, reverse-file-mapper, reverse-thinking, reverse-code |
| `LANGSMITH_TRACING` / `LANGSMITH_ENDPOINT` / `LANGSMITH_API_KEY` / `LANGSMITH_PROJECT` | all LLM-using jobs |
| `LANGCHAIN_TRACING_V2` / `LANGCHAIN_ENDPOINT` / `LANGCHAIN_API_KEY` / `LANGCHAIN_PROJECT` | reverse-code, code-generator |
| `TOKENIZERS_PARALLELISM` | LLM jobs |

## Runtime flags inside `EVENT_DATA` JSON

| Flag | Meaning |
|---|---|
| `propagate` | Continue to next stage after this job finishes |
| `resume` | Resume from saved LangGraph state |
| `is_retriggered` | Force re-run of downstream work |
| `use_k8s` | Submit k8s job instead of Pub/Sub notification |
| `single_file_mode` | Graph: one file at a time |
| `intercept` / `pr_phase` / `skip_prs` | reverse-code flow control |
| `re_ingest` / `whitelist_updated` | downloader force flags |

## Dagster translation

Env vars split into three buckets:

1. **Per-env config** (different in dev/QA/stage): topics, buckets, service URLs,
   Neo4j creds, GitHub secret server → become fields on Dagster **resources**.
2. **Per-run input** (varies per project/company): everything currently in
   `EVENT_DATA` → becomes Dagster **run-config** or op inputs.
3. **Secrets** (API keys, tokens): stay in Secret Manager; injected into the
   Dagster deployment (k8s secrets or Dagster+ secret resolver).

That split is the main reason multi-env becomes cleaner — (1) moves out of
env vars into a typed Python object per env.
