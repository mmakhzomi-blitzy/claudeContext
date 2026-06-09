# Per-job facts (from `main.py` inspection, 2026-04-20)

All jobs follow the same pattern: `python main.py`, reads `EVENT_DATA` JSON,
runs LangGraph, publishes Pub/Sub notifications, optionally submits a k8s job
for the next stage.

## archie-job-code-downloader

- Reads: GitHub via `GITHUB_SECRET_SERVER`, Neo4j credentials via company lookup
- Writes: GCS file batches, Neo4j branch copy
- Emits to: `GRAPH_CODE_TOPIC` (per batch) OR submits k8s job
  `ArchieJobType.CODE_GRAPH_GENERATOR` when `use_k8s=True`
- Key metrics: `lines_onboarded`, `files_onboarded`, `file_extensions`
- Batch size: `CODE_ONBOARDING_BATCH_SIZE`
- Retry: `@archie_exponential_retry` on download
- Runner-aware (`should_use_runner`) — may run file ops on remote runner

## archie-job-code-graph-generator

- Reads: GCS repo structure batch, Neo4j
- Writes: Neo4j graph, GCS metadata
- Emits to: `GENERATE_REVERSE_DOCUMENT_TOPIC` when all batches complete
- LLMs: `llm_gpt5_1_codex_mini` (coder), `llm_gpt5_2` (summarizer),
  `llm_claude_sonnet_4_6_adaptive` (fallback)
- Key metrics: `total_files_processed`, `total_lines_processed`
- Batch-complete detection via `graph_builder.are_other_batches_complete(...)`

## archie-job-document-generator

- Reads: input prompt from GCS, tech-spec attachments, Figma (if tech-spec has it)
- Writes: tech-spec markdown back to GCS via `AdminStorageService.upload_tech_spec`
- Emits to: `PLATFORM_EVENTS_TOPIC` (IN_PROGRESS / DONE)
- Does NOT trigger another job; downstream comes from the reverse-file-mapper branch
- Key metrics: `estimated_lines_generated`, `estimated_hours_saved` (defaults today)

## archie-job-reverse-file-mapper

- Reads: tech-spec (GCS), Neo4j, Figma info
- Writes: repo mapping, short repo structure, file schemas (all GCS)
- Emits to: `GENERATE_REVERSE_THINKING_TOPIC` via Pub/Sub

## archie-job-reverse-thinking-generator

- Reads: tech-spec, repo structure, Neo4j, env files (build info), Figma
- Writes: sorted files list, dependency map (GCS)
- Emits: submits k8s job `ArchieJobType.REVERSE_CODE_GENERATOR` (Linux) or
  `REVERSE_CODE_GENERATOR_WINDOWS` based on project target OS
- Resource sizing for next job: 2-4 CPU, 16-64GB mem, 50-250GB ephemeral storage

## archie-job-reverse-code-generator

- Reads: everything from prior stages (tech-spec, mapping, schemas, sorted files,
  dependency map, extend-PR prompt, interceptor prompt, merge-conflicts prompt)
- Writes: project guide, billing report, PR data
- Uses `Notifier` abstraction with `stage_tracker` — reports stage transitions
  (CODE_GENERATION → QA → DONE)
- Emits rich completion metadata (see `BillingReport` structure)
- Runs PR phase logic via `pr_phase` / `skip_prs` flags
- **Most metrics-rich job** — the one to showcase in a Dagster PoC

## archie-job-code-generator

- Reads: tech-spec, code structure, files list, dependency manifest
- Writes: project guide via `AdminStorageService.upload_project_guide`
- Emits to: `UPLOAD_CODE_TOPIC`
- LLMs: `llm_claude_4_sonnet_low_thinking_med_output`, `llm_gpt4_1` (fallback)
- Separate pipeline (not part of the reverse-* chain)

## Common infrastructure

- All jobs use `AdminStorageService(project_id, task_id, tech_spec_id)` for GCS
- All jobs instantiate `pubsub_v1.PublisherClient()` at import time
- All graph-aware jobs instantiate `CodeGraphBuilder` at startup
- `langsmith_tracing(...)` wraps the actual LangGraph invocation
- Retries and maintenance signal handling: `setup_maintenance_signal_handlers`
  is used in jobs that support graceful re-queue on Cloud Run eviction
