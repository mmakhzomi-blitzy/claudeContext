# Worker RQ Migration — SimpleWorker & Job Lifecycle

## Context

The worker consumed commands from Redis via raw `BLPOP`. This destructively
removed messages from the queue before processing. If the worker crashed
mid-execution (OOM, pod eviction, exception), the command was lost forever —
no retry, no dead-letter, no inspection.

Replaced with RQ's `SimpleWorker` which tracks jobs through Redis registries.
Jobs are never deleted — they move between registries until explicitly
completed or failed.

## Architecture

### Before (BLPOP)

```
Mediator enqueues → Redis List (LPUSH)
Worker BLPOP → message REMOVED → process → publish result
                 ↓
           (if crash here, message is gone)
```

### After (SimpleWorker)

```
Mediator enqueues → RQ Queue (LPUSH + Job hash)
SimpleWorker dequeues → job moves to StartedJobRegistry
  → process → FinishedJobRegistry (success)
            → FailedJobRegistry (exception)
            → stays in StartedJobRegistry (crash → requeued on next startup)
```

## Redis Key Layout

| Key | Type | Purpose |
|-----|------|---------|
| `rq:queue:{queue_name}` | List | Job IDs waiting to be processed (FIFO) |
| `rq:job:{job_id}` | Hash | Job data: func, kwargs, status, timestamps, result |
| `rq:started:{queue_name}` | Sorted Set | Jobs currently being processed (score = timestamp) |
| `rq:failed:{queue_name}` | Sorted Set | Jobs that raised exceptions |
| `rq:finished:{queue_name}` | Sorted Set | Completed jobs (TTL'd) |
| `rq:worker:{worker_id}` | Hash | Worker heartbeat and state |

## Files Changed

### Worker (`archie-client-worker`)

| File | Change |
|------|--------|
| `src/blitzy_worker/tasks.py` | **NEW** — RQ task function `execute_command(**kwargs)`. Module-level `_worker` reference set by `worker.py` before SimpleWorker starts. |
| `src/blitzy_worker/worker.py` | `run()` replaced: BLPOP loop → `SimpleWorker.work()`. Added `_rq_exception_handler` that publishes FAILED result. Uses `command_queue_base` (bare name) for RQ Queue. |

### Mediator (`archie-client-mediator`)

| File | Change |
|------|--------|
| `src/consts.py` | `WORKER_TASK_FUNC = "src.blitzy_worker.tasks.execute_command"` |
| `src/services/command_service.py` | `"func": WORKER_TASK_FUNC` in `_build_job_data()` |

## Task Function

```python
# tasks.py
_worker = None   # set by worker.py before SimpleWorker starts

def execute_command(**kwargs) -> dict:
    payload = CommandPayload(
        execution_id=kwargs.get("execution_id"),
        job_id=kwargs.get("job_id", _worker._settings.job_id),
        command=kwargs.get("command_text", ...),
        timeout_seconds=kwargs.get("timeout_seconds", ...),
        metadata=kwargs.get("metadata", {}),
    )
    result = _worker.process_payload(payload)
    return {"execution_id": ..., "status": ..., "exit_code": ...}
```

RQ resolves `"src.blitzy_worker.tasks.execute_command"` at runtime via
`importlib`. The function must be importable from the worker's Python path.

## SimpleWorker (no fork)

Standard RQ `Worker` forks a subprocess per job. We use `SimpleWorker`
(in-process, no fork) because the worker holds a persistent `BashSession`
that must survive across jobs — shell state, env vars, cwd all carry over.

## Exception Handler

When a job raises, RQ moves it to `FailedJobRegistry`. But the mediator/SDK
is polling for a result — it would hang forever. The exception handler
publishes a FAILED `ExecutionResult` so the caller gets a response:

```python
def _rq_exception_handler(self, job, *exc_info):
    result = ExecutionResult(
        execution_id=...,
        status=ExecutionStatus.FAILED,
        stderr=f"{type(exc).__name__}: {exc}",
        ...
    )
    self._result_publisher.publish(result, result.job_id)
```

## Failure Scenarios

| Scenario | Before (BLPOP) | After (SimpleWorker) |
|----------|----------------|----------------------|
| Exception during processing | Message lost | Job in FailedJobRegistry, FAILED result published to SDK |
| Worker OOMKilled | Message lost | Job in StartedJobRegistry, requeued on next worker startup |
| Worker SIGTERM (K8s) | Message lost (if mid-job) | SimpleWorker handles SIGTERM, completes current job, clean exit |
| Worker can't restart | Message lost | Job in StartedJobRegistry, recovered when pod eventually restarts |
| Result publish fails | Command ran but no result | Exception handler catches publish failure, logs it |

## Deployment Order

Worker first, then mediator. The worker must have `tasks.py` with
`execute_command` before the mediator starts enqueuing with the new
`WORKER_TASK_FUNC` path.

## Operational Commands

```bash
# Check queue depth
redis-cli LLEN rq:queue:blitzy-worker-queue-{job_id}

# View started (in-progress) jobs
redis-cli ZRANGE rq:started:blitzy-worker-queue-{job_id} 0 -1

# View failed jobs
redis-cli ZRANGE rq:failed:blitzy-worker-queue-{job_id} 0 -1

# Inspect a job
redis-cli HGETALL rq:job:{execution_id}

# Requeue all failed jobs
rq requeue --all -q blitzy-worker-queue-{job_id}
```

## Known Gaps

- **`_parse_result` stderr bug**: when worker status is FAILED, the error
  message is in stdout (JSON) but `_parse_result` reads stderr (empty).
  Consumer sees empty error message. Fix tracked in TASKS.md §16.

- **True ACK semantics**: RQ's registry approach covers most failures but
  depends on `clean_registries()` running on a live worker. If no worker
  can start (insufficient resources), jobs sit in `StartedJobRegistry`
  indefinitely. Redis Streams with XACK/XCLAIM would close this gap.
  Tracked as future consideration.
