# Blitzy Remote Command Execution - Architecture Reference

> **Purpose**: Reference for replacing direct `BashSession` usage (local subprocess)
> with `BlitzyClient` SDK → Mediator API → Worker (remote execution).
>
> **Scope of this doc**: the original HTTP-REST data plane between SDK ↔ mediator ↔ worker.
> Since this was written, two major systems were added on top:
> - **WebSocket migration**: the SDK now talks to a relay over HTTP REST, which forwards to
>   the mediator via persistent WebSocket. See `STORE_AND_FORWARD.md` for the as-built design
>   of the request relay path and outbound store-and-forward queue, and `WEBSOCKET_MIGRATION_PLAN.md`
>   for the migration history.
> - **CDP tunnel**: the Chrome DevTools Protocol passthrough used for browser-control runners.
>   See `CDP_TUNNEL_PLAN.md`.
>
> For the rewrite/replacement of mediator + relay, see `features_master.md`,
> `requirements_rewrite.md`, and `framework_evaluation.md`. The contract surface a rewrite
> must preserve is in `contract_sdk.md` and `contract_http_api.md`.

---

## Table of Contents

1. [High-Level Overview](#1-high-level-overview)
2. [The Problem: Direct BashSession Usage](#2-the-problem-direct-bashsession-usage)
3. [The Solution: Mediator + Worker + BlitzyClient](#3-the-solution-mediator--worker--blitzyclient)
4. [Component Deep Dives](#4-component-deep-dives)
5. [End-to-End Flow](#5-end-to-end-flow)
6. [BlitzyClient SDK Reference](#6-blitzyclient-sdk-reference)
7. [Mediator API Reference](#7-mediator-api-reference)
8. [Data Models and Contracts](#8-data-models-and-contracts)
9. [BashSession Consumers (What Needs Replacing)](#9-bashsession-consumers-what-needs-replacing)
10. [Replacement Strategy](#10-replacement-strategy)
11. [Gaps and Missing Pieces](#11-gaps-and-missing-pieces)
12. [File Index](#12-file-index)

---

## 1. High-Level Overview

```
BEFORE (Direct BashSession - local subprocess):
┌──────────────────────┐
│  Job Processor       │
│  (e.g. code-gen)     │
│                      │
│  BashSession.execute │──► local bash subprocess
│  (in-process)        │
└──────────────────────┘

AFTER (Remote via BlitzyClient → Mediator → Worker):
┌──────────────┐  HTTP   ┌──────────────┐  Redis   ┌──────────────┐
│ Job Processor│────────►│  Mediator    │─────────►│ Worker (K8s) │
│              │         │  (Flask API) │          │ BashSession  │
│ BlitzyClient │◄────────│  DB + Redis  │◄─────────│ (persistent) │
│ (SDK)        │  HTTP   │              │  Results │              │
└──────────────┘         └──────────────┘          └──────────────┘
```

**Core Idea**: `archie-client-mediator` + `archie-client-worker` together are the
remote equivalent of `archie-shared/common/bash.py`'s `BashSession`. The
`BlitzyClient` SDK (in `blitzy-utils-python`) provides the HTTP client that
consumers use to talk to the mediator instead of using BashSession directly.

---

## 2. The Problem: Direct BashSession Usage

### What BashSession Does

**Source**: `archie-shared/blitzy_platform_shared/common/bash.py` (~790 lines)

BashSession manages a **persistent local bash subprocess** with:

| Feature | Detail |
|---------|--------|
| Persistent process | One bash process per instance, maintains shell state (env vars, cwd) |
| Async execution | `async execute(command) -> (stdout, stderr, is_error)` |
| Sentinel-based completion | UUID sentinel strings detect when command finishes |
| Adaptive timeouts | 600s initial, 300s post-output, 6000s total |
| Output buffering | Last 200 lines, max 10MB |
| Auto-restart | Sets `_needs_restart` flag on failure, restarts on next execute |
| Process group kill | SIGINT → SIGTERM → SIGKILL on timeout |

### Key Constants (archie-shared version)

```python
DEFAULT_TIMEOUT = 6000.0              # 60 minutes total
DEFAULT_COMMAND_STARTUP_TIMEOUT = 600  # 10 minutes before first output
DEFAULT_POST_OUTPUT_TIMEOUT = 300      # 5 minutes after last output
DEFAULT_MAX_OUTPUT_SIZE = 10 * 1024 * 1024  # 10MB
DEFAULT_MAX_OUTPUT_LINES = 200
SENTINEL = "<<COMMAND_COMPLETE>>"
```

### API Surface

```python
class BashSession:
    async def start() -> Tuple[str, bool]
    async def execute(command: str) -> Tuple[str, str, bool]  # (stdout, stderr, is_error)
    async def stop() -> None
```

### Helper Functions (used by all consumers)

```python
# In bash.py - wraps execute() with formatted output for LLM tool use
async def handle_bash_tool_response(command: str, session: BashSession) -> str
    # Returns "[stdout]\n\n{stdout}\n\n[stderr]\n\n{stderr}"
    # Or "Command completed with no output."

# In utils.py - creates/restarts a session for a repo+branch
async def restart_bash_session(bash_session, repo_name, branch_name) -> Tuple[BashSession, bool]
```

---

## 3. The Solution: Mediator + Worker + BlitzyClient

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                       MAIN BLITZY SERVER                             │
│                  (registers mediator, syncs secrets)                  │
│                                                                      │
│  Admin API: GET /v1/client/entity/COMPANY/{id}                      │
│  → Returns { configuration: { url: "<mediator-url>" } }             │
└──────────────────────┬───────────────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────────────┐
        │              │                      │
        ▼              ▼                      ▼
┌──────────────┐ ┌──────────────┐  ┌──────────────────────┐
│ code-gen     │ │ file-mapper  │  │ doc-gen              │
│              │ │              │  │                      │
│ BlitzyClient │ │ BlitzyClient │  │ BlitzyClient         │
│ (SDK)        │ │ (SDK)        │  │ (SDK)                │
└──────┬───────┘ └──────┬───────┘  └──────────┬───────────┘
       │                │                      │
       └────────────────┼──────────────────────┘
                        │ HTTP REST
                        ▼
┌──────────────────────────────────────────────────────────────────────┐
│                   ARCHIE-CLIENT-MEDIATOR (Flask)                     │
│                                                                      │
│  POST /api/v1/runners              → Create K8s worker pod          │
│  DELETE /api/v1/runners/{job_id}   → Tear down worker               │
│  POST /api/v1/jobs/{id}/commands   → Submit command to Redis queue  │
│  GET /api/v1/commands/{id}/status  → Poll execution result from DB  │
│  POST /api/v1/jobs/{id}/restart-session → Send __RESTART_SESSION__  │
│                                                                      │
│  Background RQ Worker: listens on blitzy-results queue              │
│  → Updates CommandExecution DB records with worker results           │
└──────────────────────┬───────────────────────────────────────────────┘
                       │ Redis Queues
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        REDIS SERVER                                  │
│                                                                      │
│  rq:queue:blitzy-worker-queue-{job_id}  → commands TO worker        │
│  rq:queue:blitzy-results                → results FROM worker       │
└──────────────────────┬───────────────────────────────────────────────┘
                       │ BLPOP / RQ enqueue
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│                 ARCHIE-CLIENT-WORKER (K8s Pod)                       │
│                                                                      │
│  BlitzyWorker: BLPOP loop on command queue                          │
│    → CommandProcessor.execute(payload)                               │
│      → BashSession.execute(command)   ← persistent bash subprocess  │
│    → ResultPublisher.publish(result)  → enqueue to results queue    │
│                                                                      │
│  Control: __RESTART_SESSION__ → bash_session.restart()              │
└──────────────────────────────────────────────────────────────────────┘
```

### Communication Summary

| Leg | Protocol | Direction |
|-----|----------|-----------|
| BlitzyClient → Admin API | HTTP (Google Cloud Auth) | Get mediator URL for company |
| BlitzyClient → Mediator | HTTP REST (JSON, `requests`) | Submit commands, poll status |
| Mediator → Main Server | HTTP REST (`client_utils.HttpClient`, API key) | Registration, job info, secret sync |
| Worker → Main Server | HTTP REST (`client_utils.HttpClient`, API key) | Repo credentials, job info |
| Mediator → Worker | Redis Queue (RQ) | Async command dispatch |
| Worker → Mediator | Redis Queue (RQ) | Async result delivery |
| Mediator → DB | SQLAlchemy (PostgreSQL) | Persist execution records |

---

## 4. Component Deep Dives

### 4.1 BlitzyClient SDK (`blitzy-utils-python`)

**Source**: `blitzy-utils-python/blitzy_utils/blitzy_utils/blitzy_client.py`

**Role**: HTTP client SDK for consumers to interact with mediator APIs.

**Design**:
- Dependency-injected: accepts `service_client_factory` and `http_client`
- Uses `ServiceClient` (admin API with Google Cloud auth) for URL lookup
- Uses `requests` (via `_RequestsAdapter`) for direct mediator HTTP calls
- Provides both class-based (`BlitzyClient`) and module-level convenience functions

**URL Resolution Flow**:
```
BlitzyClient.create_runner("company-123", "job-456")
  → get_client_url("company-123")
    → ServiceClient.get("admin", "/v1/client/entity/COMPANY/company-123")
    → Response: { configuration: { url: "https://mediator.example.com" } }
  → POST https://mediator.example.com/api/v1/runners
```

### 4.2 Mediator (`archie-client-mediator`)

**Role**: HTTP API gateway + job orchestrator + status tracker

**Layered Architecture**:
```
API Routes → Services → Repositories → PostgreSQL
                ↓
          RedisQueueService → Redis
                ↓
          KubernetesService → K8s API
```

**Key Design Patterns**:
- Interface segregation (all services/repos have `I*` interfaces)
- Dependency injection
- Repository pattern for DB access

### 4.3 Worker (`archie-client-worker`)

**Role**: Command executor in isolated K8s pod

**Worker BashSession Constants** (different from shared version):
```python
DEFAULT_TIMEOUT = 300                  # 5 min total (vs 60 min shared)
DEFAULT_COMMAND_STARTUP_TIMEOUT = 30   # 30s (vs 10 min shared)
DEFAULT_POST_OUTPUT_TIMEOUT = 10       # 10s (vs 5 min shared)
DEFAULT_MAX_OUTPUT_SIZE = 1024 * 1024  # 1MB (vs 10MB shared)
DEFAULT_MAX_OUTPUT_LINES = 1000        # 1000 lines (vs 200 shared)
SENTINEL = "___BLITZY_SENTINEL_END___"
```

**Key Design Patterns**:
- Template method (before_job/after_job/on_error hooks)
- Strategy pattern (CommandProcessor bridges async/sync)
- Control command pattern (`__RESTART_SESSION__`)

---

## 5. End-to-End Flow

### Step 1: Create Runner (one-time per job)

```
BlitzyClient.create_runner("company-123", "job-456")
  ↓
POST /api/v1/runners { blitzy_job_id: "job-456", runner_config: {...} }
  ↓
Mediator:
  1. KubernetesService.create_deployment() → K8s pod with worker image
  2. RedisQueueService.create_queue()      → "blitzy-worker-queue-job-456"
  3. JobServerRepository.create()          → DB record
  4. CallbackQueueRepository.create()      → DB link job → queue
  ↓
Response: 201 { runner_id, deployment_name, queue_name }
```

### Step 2: Submit Command

```
BlitzyClient.create_job_command("company-123", "job-456", "ls -la")
  ↓
POST /api/v1/jobs/job-456/commands { command_text: "ls -la", timeout_seconds: 300 }
  ↓
Mediator (CommandService.submit_command):
  1. Lookup queue for job-456
  2. Create CommandExecution record (status=QUEUED) in DB
  3. Build RQ job: { func: "execute_command", kwargs: { execution_id, command_text, ... } }
  4. RedisQueueService.enqueue_job("blitzy-worker-queue-job-456", job_data)
  ↓
Response: 201 { execution_id: "exec-789" }
```

### Step 3: Worker Executes

```
Worker BLPOP loop dequeues from rq:queue:blitzy-worker-queue-job-456
  ↓
Job.fetch(job_id) → extract kwargs → build CommandPayload
  ↓
before_job() → log "job_received"
  ↓
CommandProcessor.execute(payload):
  BashSession.execute("ls -la") → (stdout, stderr, is_error)
  Map to ExecutionResult(status=COMPLETED/FAILED/TIMEOUT)
  ↓
after_job() → log "job_completed"
  ↓
ResultPublisher.publish(result) → enqueue to rq:queue:blitzy-results
```

### Step 4: Mediator Processes Result

```
Background RQ Worker dequeues from blitzy-results
  ↓
Calls job_processor.process_job() → command_result_handler.handle_command_result()
  ↓
Updates CommandExecution in DB: status, exit_code, stdout, stderr, timestamps
```

### Step 5: Poll Status

```
BlitzyClient.get_command_status("company-123", "exec-789")
  ↓
GET /api/v1/commands/exec-789/status
  ↓
Mediator reads from DB → returns full execution details
  ↓
Response: {
  execution_id, status: "completed", exit_code: 0,
  stdout: "file1.txt\nfile2.txt\n", stderr: "", ...
}
```

### Session Restart Flow

```
POST /api/v1/jobs/job-456/restart-session
  ↓
Mediator internally: submit_command(job_id, { command_text: "__RESTART_SESSION__" })
  ↓
Worker detects control command → bash_session.restart() → stop() + start()
  ↓
Publishes result: status=COMPLETED or FAILED
```

---

## 6. BlitzyClient SDK Reference

**Source**: `blitzy-utils-python/blitzy_utils/blitzy_utils/blitzy_client.py`

### Class: BlitzyClient

```python
class BlitzyClient:
    def __init__(
        self,
        service_client_factory=None,  # For admin API (Google Cloud auth)
        http_client=None,             # For mediator HTTP (default: requests)
    )
```

### Methods

#### `get_client_url(company_id: str) -> str`
Resolves the mediator URL for a company via admin API.
- Calls: `GET /v1/client/entity/COMPANY/{company_id}` on admin service
- Returns: `configuration.url` from response

#### `create_runner(company_id, job_id, cpu_request="100", cpu_limit="500", memory_request="256", memory_limit="512") -> dict`
Creates a K8s worker pod + Redis queue.
- Calls: `POST {mediator_url}/api/v1/runners`
- Returns: Full API response dict

#### `delete_runner(company_id, job_id) -> dict`
Tears down worker pod and cleans up queue.
- Calls: `DELETE {mediator_url}/api/v1/runners/{job_id}`
- Returns: API response dict

#### `create_job_command(company_id, job_id, command_text, timeout_seconds=300, working_directory=None, environment_variables=None, priority="normal", callback_url=None, metadata=None) -> str`
Submits a command for execution.
- Calls: `POST {mediator_url}/api/v1/jobs/{job_id}/commands`
- Returns: `execution_id` string

#### `get_command_status(company_id, command_id) -> dict`
Polls execution status.
- Calls: `GET {mediator_url}/api/v1/commands/{command_id}/status`
- Returns: Full status dict with stdout, stderr, exit_code, status, timestamps

#### `execute_command(company_id, job_id, command_text, ...) -> dict`
**Added since first draft of this doc.** Submits a command and polls until terminal status. Replaces the manual `create_job_command` + poll loop.
- Calls: `POST .../commands` then polls `GET .../commands/{id}/status` with backoff
- Returns: Final command status dict
- Source: `blitzy-utils-python/blitzy_utils/blitzy_client.py:650-679`

#### `execute_bash_tool(company_id, job_id, command_text, ...) -> str`
**Added since first draft of this doc.** LLM-tool-compatible wrapper around `execute_command` that formats output as `[stdout]\n\n{stdout}\n\n[stderr]\n\n{stderr}` (drop-in for the legacy `handle_bash_tool_response`).
- Source: `blitzy-utils-python/blitzy_utils/blitzy_client.py:681-717`

#### `restart_session(company_id, job_id) -> dict`
**Added since first draft of this doc.** Restarts the worker's bash session via the mediator's `restart-session` endpoint.
- Calls: `POST {mediator_url}/api/v1/jobs/{job_id}/restart-session`
- Source: `blitzy-utils-python/blitzy_utils/blitzy_client.py:535-550`

#### `wait_for_ready(company_id, job_id, timeout=60) -> dict`
**Added since first draft of this doc.** Polls runner status until `READY` or timeout. Replaces the manual create+poll loop consumers had to write.
- Source: `blitzy-utils-python/blitzy_utils/blitzy_client.py:618-648`

#### `push_to_mediator(company_id, event, payload) -> Any`
**Added since first draft of this doc.** Direct Socket.IO emit to the mediator for out-of-band signals (used for MCP asset push and similar non-HTTP control events).
- Source: `blitzy-utils-python/blitzy_utils/blitzy_client.py:569-596`

### Module-Level Convenience Functions

All class methods are also available as module-level functions using a lazy default instance:
```python
from blitzy_utils.blitzy_client import (
    get_client_url, create_runner, delete_runner,
    create_job_command, get_command_status,
)
```

---

## 7. Mediator API Reference

### Runner Management

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/runners` | POST | Create runner (K8s pod + Redis queue) |
| `/api/v1/runners/{job_id}` | DELETE | Tear down runner and cleanup |

### Command Execution

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/jobs/{job_id}/commands` | POST | Submit command for execution |
| `/api/v1/commands/{execution_id}/status` | GET | Poll execution status |
| `/api/v1/jobs/{job_id}/restart-session` | POST | Restart worker bash session |

### Registration and Health

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/register` | POST | Register mediator with main server |
| `/api/v1/secrets/sync` | POST | Trigger Vault secret sync |
| `/v1/health-check` | GET | Health check |
| `/v1/uptime-check` | GET | Uptime check |

---

## 8. Data Models and Contracts

### Command Submission Request

```json
{
  "command_text": "echo 'Hello World'",
  "timeout_seconds": 300,
  "working_directory": "/tmp/blitzy/repo/main",
  "environment_variables": { "KEY": "value" },
  "priority": "normal",
  "callback_url": null,
  "metadata": { "source": "code-gen" }
}
```

### Command Submission Response

```json
{
  "execution_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

### Command Status Response

```json
{
  "execution_id": "550e8400-e29b-41d4-a716-446655440000",
  "blitzy_job_id": "job-123",
  "server_id": "server-456",
  "command_text": "echo 'Hello World'",
  "status": "completed",
  "exit_code": 0,
  "stdout": "Hello World\n",
  "stderr": "",
  "timeout_seconds": 300,
  "submitted_at": "2024-01-15T10:30:00.000000+00:00",
  "started_at": "2024-01-15T10:30:01.000000+00:00",
  "completed_at": "2024-01-15T10:30:02.000000+00:00",
  "metadata": null
}
```

### Status Values

| Status | Description |
|--------|-------------|
| `QUEUED` | Submitted, waiting for worker pickup |
| `COMPLETED` | Finished successfully (exit_code=0) |
| `FAILED` | Finished with error (exit_code!=0) |
| `TIMEOUT` | Exceeded timeout limit |
| `PENDING` | Reserved (also defined in enum at `src/api/models.py:55-63`) |
| `RUNNING` | Reserved — see Gap #3 in §11; mediator enum advertises this but worker never emits it |
| `CANCELLED` | Reserved (also defined in enum) |

> **Note**: status values are **UPPERCASE strings**. Earlier drafts of this doc showed lowercase — that was wrong.

### Worker Internal Models

```python
class CommandPayload:       # Input to worker
    execution_id: UUID
    job_id: str
    command: str
    timeout_seconds: int = 300
    metadata: Dict[str, Any] = {}

class ExecutionResult:      # Output from worker
    execution_id: UUID
    job_id: str
    status: ExecutionStatus  # COMPLETED | FAILED | TIMEOUT
    exit_code: int
    stdout: str
    stderr: str
    started_at: datetime
    completed_at: datetime
```

---

## 9. BashSession Consumers (What Needs Replacing)

### Current Consumers Using Direct BashSession

| Consumer Repo | File | Import |
|---------------|------|--------|
| `archie-job-reverse-code-generator` | `lib/blitzy/helper.py` | `handle_bash_tool_response`, `restart_bash_session` |
| `archie-job-reverse-file-mapper` | `lib/reverse_mapper/helper.py` | `restart_bash_session`, `handle_bash_tool_response` |
| `archie-job-reverse-document-generator` | `lib/reverse_document/helper.py` | `handle_bash_tool_response`, `restart_bash_session` |

### Identical Usage Pattern in All Three

```python
from blitzy_platform_shared.common.bash import handle_bash_tool_response
from blitzy_platform_shared.common.bash import restart_bash_session

class SomeJobHelper:
    def __init__(self):
        self.bash_session = None  # Step 1: Initialize as None

    async def setup(self):
        # Step 2: Create session (points to local repo+branch working dir)
        self.bash_session, _ = await restart_bash_session(
            bash_session=self.bash_session,
            repo_name=self.repo_name,
            branch_name=self.branch_name,
        )

    async def handle_tool_call(self, command, restart=False):
        if restart:
            # Step 3a: Restart when LLM requests it
            self.bash_session, _ = await restart_bash_session(
                bash_session=self.bash_session,
                repo_name=self.repo_name,
                branch_name=self.branch_name,
            )
            return "Bash session restarted successfully."
        else:
            # Step 3b: Execute command
            return await handle_bash_tool_response(
                command=command, session=self.bash_session
            )
```

### What handle_bash_tool_response Returns (LLM-facing format)

```python
# If stdout and stderr present:
"[stdout]\n\n{stdout}\n\n[stderr]\n\n{stderr}"

# If only stdout:
"[stdout]\n\n{stdout}"

# If no output:
"Command completed with no output."
```

---

## 10. Replacement Strategy

### Mapping: BashSession API → BlitzyClient API

| Current (BashSession) | Replacement (BlitzyClient) |
|------------------------|---------------------------|
| `restart_bash_session(session, repo, branch)` | `create_runner(company_id, job_id)` (first time) |
| `session.execute(command)` | `create_job_command(...)` + poll `get_command_status(...)` |
| `handle_bash_tool_response(cmd, session)` | Submit + poll + format output identically |
| `session.stop()` | `delete_runner(company_id, job_id)` |
| Restart (LLM `restart=True`) | `POST /api/v1/jobs/{id}/restart-session` via mediator |

### What a Drop-in Wrapper Looks Like

```python
import time
from blitzy_utils.blitzy_client import BlitzyClient

TERMINAL_STATUSES = {"completed", "failed", "timeout"}

class RemoteBashSession:
    """Drop-in replacement for direct BashSession via BlitzyClient."""

    def __init__(self, client: BlitzyClient, company_id: str, job_id: str):
        self._client = client
        self._company_id = company_id
        self._job_id = job_id

    def execute(self, command: str, timeout: int = 300) -> tuple[str, str, bool]:
        """Submit command, poll until done, return (stdout, stderr, is_error)."""
        exec_id = self._client.create_job_command(
            self._company_id, self._job_id, command, timeout_seconds=timeout
        )
        # Poll with backoff
        while True:
            status = self._client.get_command_status(self._company_id, exec_id)
            if status["status"] in TERMINAL_STATUSES:
                break
            time.sleep(1)

        stdout = status.get("stdout", "")
        stderr = status.get("stderr", "")
        is_error = status["status"] != "completed"
        return stdout, stderr, is_error

    def restart(self) -> tuple[str, bool]:
        """Restart the remote worker's bash session."""
        exec_id = self._client.create_job_command(
            self._company_id, self._job_id, "__RESTART_SESSION__", timeout_seconds=30
        )
        while True:
            status = self._client.get_command_status(self._company_id, exec_id)
            if status["status"] in TERMINAL_STATUSES:
                break
            time.sleep(1)
        return status.get("stdout", ""), status["status"] != "completed"


def handle_bash_tool_response_remote(command: str, session: RemoteBashSession) -> str:
    """Drop-in replacement for handle_bash_tool_response."""
    stdout, stderr, is_error = session.execute(command)
    output_parts = []
    if stdout:
        output_parts.append(f"[stdout]\n\n{stdout}")
    if stderr:
        output_parts.append(f"[stderr]\n\n{stderr}")
    return "\n\n".join(output_parts) if output_parts else "Command completed with no output."
```

---

## 11. Gaps and Missing Pieces

### High Severity

| # | Gap | Detail |
|---|-----|--------|
| 1 | ~~**No polling/wait logic in BlitzyClient**~~ — **RESOLVED**. `BlitzyClient.execute_command()` provides built-in submit + poll with backoff; `BlitzyClient.wait_for_ready()` polls runner status until READY. See `blitzy-utils-python/blitzy_utils/blitzy_client.py:618-679`. |
| 2 | ~~**No `restart_session` method in BlitzyClient**~~ — **RESOLVED**. `BlitzyClient.restart_session()` exists at `blitzy-utils-python/blitzy_utils/blitzy_client.py:535-550`. |

### Medium Severity

| # | Gap | Detail |
|---|-----|--------|
| 3 | **No `running` status from worker** | Worker only publishes final results. DB record stays `QUEUED` until completion. Polling consumers can't distinguish "worker hasn't started" from "worker is executing". **Inconsistency**: mediator's `CommandExecutionStatus` enum at `src/api/models.py:55-63` includes `RUNNING` but worker's `ExecutionStatus` enum (`archie-client-worker/src/blitzy_worker/models.py:20-35`) only has `COMPLETED`/`FAILED`/`TIMEOUT`. Mediator API schema advertises a status the worker never emits. |
| 4 | **No streaming/real-time output** | BashSession streams output in real-time. Mediator is poll-only. Long-running commands give no intermediate feedback. No WebSocket/SSE support. |
| 5 | **Timeout mismatch** | Shared BashSession: 60-min max. Worker BashSession: 5-min default. Worker does read `timeout_seconds` from payload but the default mismatch could cause issues if consumers don't pass explicit timeouts. |
| 6 | **Output size difference** | Shared: 10MB, 200 lines. Worker: 1MB, 1000 lines. Large output may be truncated differently. |
| 7 | **Error type mapping** | BashSession returns `is_error: bool`. Mediator returns `status` enum + `exit_code`. The `is_error = (status != "completed")` mapping needs to be standardized and documented. |
| 8 | **Authentication undefined** | BlitzyClient uses `requests` (no auth) for mediator HTTP calls but `ServiceClient` (Google Cloud auth) for admin API. If the mediator requires auth headers (e.g., `x-client-key`), BlitzyClient doesn't set them. |
| 9 | **`working_directory` and `environment_variables` params in `create_job_command`** | BlitzyClient accepts these but the mediator's `CommandService._extract_*` methods don't extract or forward them. These fields are set at pod creation time (runner config), not per-command. |
| 10 | **Runner lifecycle ownership** | Currently the main server triggers runner creation. If job processors replace BashSession, they need runner create/destroy integrated into their own lifecycle (setup/teardown). |

### Low Severity

| # | Gap | Detail |
|---|-----|--------|
| 11 | **`handle_bash_tool_response` format contract** | The `[stdout]\n\n...\n\n[stderr]\n\n...` format is consumed by LLMs. The remote replacement must produce identical formatting. |
| 12 | **Secret/env sync for remote workers** | Local BashSession inherits parent env vars. Remote workers need explicit ConfigMap injection or `/secrets/sync`. |
| 13 | **`get_client_url` called on every operation** | Each `create_runner`, `create_job_command`, etc. calls `get_client_url` first (admin API round-trip). Should cache the URL per company to reduce latency. |

### Recommendations

1. **Add to BlitzyClient SDK**:
   - `execute_command_and_wait(company_id, job_id, command, ...)` - submit + poll with configurable backoff
   - `restart_session(company_id, job_id)` - wraps restart endpoint
   - URL caching for `get_client_url`

2. **Add `RUNNING` status update** to worker (publish intermediate status when execution starts)

3. **Standardize timeout config** - ensure worker respects per-command `timeout_seconds` and document max values

4. **Build `RemoteBashSession` wrapper** (as shown in Section 10) to provide identical interface to existing consumers

---

## 12. File Index

### BlitzyClient SDK (`blitzy-utils-python`)

| File | Purpose |
|------|---------|
| `blitzy_utils/blitzy_utils/blitzy_client.py` | `BlitzyClient` class + module-level convenience functions |
| `blitzy_utils/blitzy_utils/service_client.py` | `ServiceClient` - authenticated HTTP to Cloud Run services |
| `blitzy_utils/blitzy_utils/test/test_blitzy_client.py` | Unit tests for BlitzyClient |

### Shared Client Utils (`blitzy-utils-python/blitzy_client_utils`)

| File | Purpose |
|------|---------|
| `client_utils/http_client.py` | `HttpClient` - API key-authenticated HTTP client for main server communication. Shared by mediator and worker. |
| `pyproject.toml` | Package config: `blitzy-client-utils` |
| `Makefile` | Build/deploy automation |

> **Note**: `HttpClient` was extracted from `archie-client-mediator/src/utils/http_client.py` into this
> shared library so both mediator and worker can use the same client to talk to the main Blitzy server
> with API key (`x-client-key`) authentication. The mediator now imports via `from client_utils.http_client import HttpClient`.

### BashSession Definition (`archie-shared`)

| File | Purpose |
|------|---------|
| `blitzy_platform_shared/common/bash.py` | `BashSession` class + `handle_bash_tool_response()` + `restart_bash_session()` |

### Mediator (`archie-client-mediator`)

| File | Purpose |
|------|---------|
| `main.py` | Flask app entry, RQ worker subprocess, signal handlers |
| `src/api/routes/commands.py` | Command submit + status endpoints |
| `src/api/routes/runners.py` | Runner CRUD endpoints |
| `src/api/routes/registration.py` | Server registration |
| `src/api/routes/secrets.py` | Vault secret sync |
| `src/services/command_service.py` | Command orchestration (submit, status, restart) |
| `src/services/runner_service.py` | Runner lifecycle (K8s + Redis + DB) |
| `src/services/kubernetes_service.py` | K8s deployment create/delete |
| `src/services/redis_queue_service.py` | Redis queue operations (enqueue/dequeue) |
| `src/services/blitzy_service.py` | Main server registration + secret sync |
| `src/workers/job_processor.py` | RQ job routing for results queue |
| `src/workers/command_result_handler.py` | Process results from worker → update DB |
| `src/repositories/command_execution_repository.py` | CommandExecution CRUD |
| `src/repositories/callback_queue_repository.py` | CallbackQueue CRUD (job → queue mapping) |
| `src/repositories/job_server_repository.py` | JobServer CRUD |
| `src/utils/kube_client_provider.py` | K8s client factory |
| `src/utils/redis_client_provider.py` | Redis client factory |
| `src/consts.py` | Environment variable constants |
| `src/api/models.py` | Request/response Pydantic models |

### Worker (`archie-client-worker`)

| File | Purpose |
|------|---------|
| `src/blitzy_worker/main.py` | Entry point, signal handling, graceful shutdown |
| `src/blitzy_worker/worker.py` | Main BLPOP loop, job orchestration, template method hooks |
| `src/blitzy_worker/processor.py` | Async→sync bridge for BashSession execution |
| `src/blitzy_worker/bash_session.py` | Persistent bash subprocess management (worker's local copy) |
| `src/blitzy_worker/models.py` | `CommandPayload`, `ExecutionResult`, `ExecutionStatus` |
| `src/blitzy_worker/result_publisher.py` | Publishes results to Redis results queue |
| `src/blitzy_worker/config.py` | Settings from environment variables |

### Consumers (need BashSession → BlitzyClient replacement)

| File | Consumer |
|------|----------|
| `archie-job-reverse-code-generator/lib/blitzy/helper.py` | Code generation jobs |
| `archie-job-reverse-file-mapper/lib/reverse_mapper/helper.py` | File mapping jobs |
| `archie-job-reverse-document-generator/lib/reverse_document/helper.py` | Doc generation jobs |
