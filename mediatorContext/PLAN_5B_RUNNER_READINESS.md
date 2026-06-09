# Plan: Task 5b — Runner Readiness Signal

## Context

After `POST /runners` creates a K8s deployment, the worker pod starts, clones the repo (startup pipeline), and enters the BLPOP loop. Currently there's no way for consumers to know when the clone is done — K8s deployment `Ready` only means the container is running. Consumers need to wait for `READY` status before sending commands.

## Architecture

```
Consumer (GCP)          Main Server           Mediator              Worker
   │                       │                     │                     │
   ├─ create_runner() ────►│── POST /runners ───►│── create K8s pod ──►│
   │                       │                     │                     ├─ startup.execute()
   │                       │                     │                     │  (clone repo...)
   │  poll                 │                     │                     │
   ├─ wait_for_ready() ──►│── GET /runners/{id}─►│                     │
   │◄─ DEPLOYING ──────────│                     │                     │
   │                       │                     │    RQ: server_ready │
   │  poll                 │                     │◄────────────────────│ clone done!
   ├─ wait_for_ready() ──►│── GET /runners/{id}─►│  (status=READY)    │
   │◄─ READY ──────────────│                     │                     ├─ enter BLPOP
   │                       │                     │                     │
   ├─ execute_command() ──►│── POST /commands ──►│── LPUSH ──────────►│
```

- **Worker → Mediator**: via existing RQ results queue (no HTTP call from worker)
- **Consumer → Main Server → Mediator**: via `GET /runners/{job_id}` (existing proxy pattern)

## Implementation

### 5b-i: Mediator — RQ handler + GET endpoint

#### A. New RQ handler: `handle_server_ready()`

**File**: `src/workers/command_result_handler.py` (or new `src/workers/server_ready_handler.py`)

- Receives `job_id` from RQ job data
- Looks up `JobServer` by `blitzy_job_id`
- Updates status `DEPLOYING → READY` via `JobServerRepository.update_status()` (auto-sets `ready_at`)
- Finds `CallbackQueue` records with `callback_type=SERVER_READY` and `status=PENDING`, transitions to `SENT`

#### B. Register handler in job processor

**File**: `src/workers/job_processor.py`

- Add `"server_ready": handle_server_ready` to `HANDLERS` dict
- Add routing case in `process_job()` for `type == "server_ready"`

#### C. GET endpoint for runner status

**File**: `src/api/routes/runners.py`

- `GET /runners/<job_id>` — calls existing `service.get_runner_by_job_id(job_id)`, returns status
- Uses `_create_runner_service()` factory (existing pattern)

#### D. Swagger + models

**File**: `swagger.yaml` — add `GET /runners/{job_id}` endpoint + `RunnerStatusResponse` schema

**File**: `src/api/models.py` — auto-generated from swagger (just need swagger changes)

#### E. Interface update

**File**: `src/services/interfaces/i_runner_service.py` — add `mark_runner_ready(blitzy_job_id: str) -> Any`

**File**: `src/services/runner_service.py` — implement `mark_runner_ready()`

---

### 5b-ii: Worker — Enqueue readiness signal via RQ

**Files**: `archie-client-worker/src/blitzy_worker/result_publisher.py`, `worker.py`, `main.py`

After `startup.execute()` and `BlitzyWorker()` creation, before `worker.run()`:

- `ResultPublisher.publish_signal()` — enqueues control signals to result queue using same RQ pattern
- `BlitzyWorker.signal_ready()` — calls `publish_signal("server_ready", job_id=...)`, raises on failure
- `main.py` — calls `worker.signal_ready()` before entering BLPOP loop
- No HTTP call needed, no mediator URL discovery needed

---

### 5b-iii: SDK — `wait_for_ready()` in BlitzyClient

**File**: `blitzy-utils-python/blitzy_utils/blitzy_utils/blitzy_client.py`

- `get_runner_status(company_id, job_id)` — `GET /api/v1/runners/{job_id}` (calls main server, which proxies to mediator)
- `wait_for_ready(company_id, job_id, timeout=120, poll_interval=2.0)` — polls `get_runner_status()` with backoff until `READY` or timeout
- Module-level convenience wrappers

---

## Files to modify

| File | Repo | Change |
|------|------|--------|
| `src/workers/server_ready_handler.py` | mediator | **CREATE** — `handle_server_ready()` |
| `src/workers/job_processor.py` | mediator | Register `server_ready` handler |
| `src/services/runner_service.py` | mediator | Add `mark_runner_ready()` |
| `src/services/interfaces/i_runner_service.py` | mediator | Add to interface |
| `src/api/routes/runners.py` | mediator | Add `GET /runners/{job_id}`, fix enum imports |
| `src/api/routes/commands.py` | mediator | Fix enum imports (`Status3/4` → named enums) |
| `swagger.yaml` | mediator | Add GET endpoint + response schema + named enum schemas |
| `src/blitzy_worker/result_publisher.py` | worker | Add `publish_signal()` method |
| `src/blitzy_worker/worker.py` | worker | Add `signal_ready()` method |
| `src/blitzy_worker/main.py` | worker | Call `worker.signal_ready()` before BLPOP |
| `blitzy_utils/blitzy_client.py` | blitzy-utils | Add `get_runner_status()`, `wait_for_ready()` |

## Verification

1. Worker enqueues `server_ready` job after startup — check mediator RQ worker logs
2. `JobServer` status flips from `DEPLOYING` to `READY` with `ready_at` set
3. `GET /runners/{job_id}` returns current status
4. End-to-end: create runner → worker clones → RQ signal → GET returns READY
