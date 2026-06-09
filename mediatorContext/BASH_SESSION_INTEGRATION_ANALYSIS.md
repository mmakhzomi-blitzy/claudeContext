# BashSession → BlitzyClient Integration Analysis

> **Scope**: Deep dive into `restart_bash_session`, how `reverse-document-generator`
> uses it, and the concrete changes needed to integrate the BlitzyClient SDK.
>
> **Companion doc**: See `ARCHITECTURE_REFERENCE.md` for the full system architecture.

---

## Table of Contents

1. [restart_bash_session - Source Analysis](#1-restart_bash_session---source-analysis)
2. [reverse-document-generator Usage](#2-reverse-document-generator-usage)
3. [Code Flow Through the LangGraph Pipeline](#3-code-flow-through-the-langgraph-pipeline)
4. [BlitzyClient SDK - Current State](#4-blitzyclient-sdk---current-state)
5. [Integration Plan](#5-integration-plan)
6. [SDK Gaps to Fill Before Integration](#6-sdk-gaps-to-fill-before-integration)
7. [Consumer-Side Changes (reverse-document-generator)](#7-consumer-side-changes-reverse-document-generator)

---

## 1. restart_bash_session - Source Analysis

**File**: `archie-shared/blitzy_platform_shared/common/utils.py` (lines 631-640)

```python
async def restart_bash_session(
    bash_session: BashSession, repo_name: str, branch_name: str
) -> Tuple[BashSession, bool]:
    if bash_session:
        await bash_session.stop()
    bash_session = BashSession(
        working_directory=get_cwd(repo_name=repo_name, branch_name=branch_name)
    )
    start_result, is_error = await bash_session.start()
    if is_error:
        logger.error(f"Bash session start Error: {start_result}")
    else:
        logger.info(f"Bash session restarted: {start_result}")
    return bash_session, is_error
```

### What It Does (step by step)

1. **Stop existing session** - If a session already exists, calls `await bash_session.stop()` which:
   - Cancels any running command
   - Closes stdin (sends EOF)
   - Sends SIGTERM → SIGKILL to the bash process
   - Drains stdout/stderr
   - Cleans up asyncio tasks

2. **Resolve working directory** - Calls `get_cwd(repo_name, branch_name)` from `blitzy_utils.disk`
   - Returns a path like `/tmp/blitzy/{repo_name}/{sanitized_branch_name}`
   - This is where the repo was downloaded to by `download_repository_to_disk()`

3. **Create new BashSession** - Instantiates `BashSession(working_directory=...)` with:
   - `timeout=6000.0` (60 minutes)
   - `output_delay=0.1` (100ms poll)
   - `max_output_size=10MB`
   - `max_output_lines=200`

4. **Start the session** - Calls `await bash_session.start()` which:
   - Spawns `bash` subprocess with PIPE for stdin/stdout/stderr
   - Sets `PS1=''` (no prompt), `TERM=dumb`, `set +o pipefail`
   - `cd`s into the working directory
   - Returns `(success_message, is_error)`

5. **Return** - Returns `(bash_session, is_error)` tuple

### Dependencies

| Dependency | Source | Purpose |
|------------|--------|---------|
| `BashSession` | `blitzy_platform_shared.common.bash` | The class being created |
| `get_cwd()` | `blitzy_utils.disk` | Resolves `/tmp/blitzy/{repo}/{branch}` path |
| `logger` | `blitzy_utils.logger` | Structured logging |

---

## 2. reverse-document-generator Usage

**File**: `archie-job-reverse-document-generator/lib/reverse_document/helper.py`

### Imports (lines 27, 40-48)

```python
from blitzy_platform_shared.common.bash import handle_bash_tool_response, restart_bash_session
```

### Initialization (line 222)

```python
class ReverseDocumentHelper:
    def __init__(self, ...):
        # ...
        self.bash_session = None   # No session at construction time
```

`bash_session` starts as `None`. It gets created during `setup()` once the repo
is downloaded to disk. Metadata available on `self` for the SDK:
- `self.company_id` (line 208)
- `self.repo_name` (line 214)
- `self.branch_name` (line 215)

### Call Site 1: `setup()` - One-time initialization (lines 362-366)

```python
async def setup(self, state: ReverseDocumentState) -> Dict[str, Any]:
    # ... state initialization, root folder fetch ...

    download_repository_to_disk(         # Download repo to disk first
        repo_name=self.repo_name,
        branch_name=self.branch_name,
        # ...
    )

    self.bash_session, _ = await restart_bash_session(   # THEN create session
        bash_session=self.bash_session,                  # None on first call
        repo_name=self.repo_name,
        branch_name=self.branch_name,
    )

    return get_state(state=state)
```

**Purpose**: After downloading the repository to local disk, create the initial
bash session so the LLM can run shell commands (grep, find, cat, etc.) against
the codebase. Runs **once** at the start of the LangGraph pipeline.

**Key detail**: `self.bash_session` is `None` here, so the `if bash_session: stop()`
branch is skipped - it only creates and starts a new session.

### Call Site 2: `gather_context()` - LLM-triggered (lines 450-469)

```python
@archie_exponential_retry()
async def gather_context(self, state: ReverseDocumentState) -> Dict[str, Any]:
    # ... build messages, call LLM ...

    while len(response.tool_calls):
        for tool_call in tool_calls:
            tool_name = tool_call["name"]
            args = tool_call.get("args", None)

            if tool_name == ANTHROPIC_BASH_TOOL_NAME:
                if not args:
                    tool_result = "Invalid tool call..."
                else:
                    restart: bool = args.get("restart", False)
                    command: str = args.get("command", "")

                    if not command and not restart:
                        tool_result = "Invalid tool call..."

                    if restart:                                    # LLM requests restart
                        self.bash_session, _ = await restart_bash_session(
                            bash_session=self.bash_session,
                            repo_name=self.repo_name,
                            branch_name=self.branch_name,
                        )
                        tool_result = "Bash session restarted successfully."
                    else:                                          # LLM runs a command
                        tool_result = await handle_bash_tool_response(
                            command=command, session=self.bash_session
                        )

                tool_message = ToolMessage(
                    content=tool_result,
                    name=ANTHROPIC_BASH_TOOL_NAME,
                    tool_call_id=tool_call["id"],
                )
```

**Purpose**: The **search LLM** explores the codebase to gather context for each
tech spec section. It can use bash for commands like `grep -r "auth" src/`,
`find . -name "*.py"`, `cat src/config.py`, etc. If the session gets stuck,
the LLM can request a restart via `{ "restart": true }`.

**This node is called once per section** in GENERATE mode (loops via
`gather_context → document_section → gather_context → ...`).

### Call Site 3: `process_section()` - LLM-triggered (lines 544-563)

```python
async def process_section(self, state, messages, section_heading, tools_list, llm):
    # ... LLM tool call loop - identical pattern to gather_context ...

    if tool_name == ANTHROPIC_BASH_TOOL_NAME:
        if not args:
            tool_result = "Invalid tool call..."
        else:
            restart: bool = args.get("restart", False)
            command: str = args.get("command", "")
            if not command and not restart:
                tool_result = "Invalid tool call..."
            if restart:
                self.bash_session, _ = await restart_bash_session(
                    bash_session=self.bash_session,
                    repo_name=self.repo_name,
                    branch_name=self.branch_name,
                )
                tool_result = "Bash session restarted successfully."
            else:
                tool_result = await handle_bash_tool_response(
                    command=command, session=self.bash_session
                )
```

**Purpose**: Called by `summarize_changes()` (line 902). The **summarizer LLM**
can also run bash commands while generating documentation. Exact same bash
tool handling pattern as `gather_context`.

### Where process_section Is Invoked

```python
# summarize_changes() at line 902
document_sections = await self.process_section(
    state=state,
    messages=messages,
    section_heading=section_heading,
    tools_list=summarizer_node_tools,
    llm=llm_claude_opus_4_5_thinking.bind_tools(
        tools=summarizer_llm_tools, parallel_tool_calls=False
    ),
)
```

---

## 3. Code Flow Through the LangGraph Pipeline

```
ReverseDocumentHelper.__init__()
  ├─ self.bash_session = None
  ├─ self.company_id = job_metadata["company_id"]
  ├─ self.repo_name = job_metadata["repo_name"]
  ├─ self.branch_name = job_metadata["branch_name"]
  └─ create_graph() builds LangGraph state machine

LangGraph execution:

  START
    │
    ▼
  setup()
    ├─ Initialize state (section prompts, root folder, etc.)
    ├─ download_repository_to_disk(repo, branch)
    ├─ restart_bash_session(None, repo, branch)  ◄── CREATE session
    └─ return state
    │
    ├── [GENERATE mode] ──────────────────────────┐
    │                                              │
    ▼                                              │
  gather_context()    ◄── loops per section        │
    ├─ Build system/human messages for search LLM  │
    ├─ LLM tool call loop:                         │
    │   ├─ bash {command: "grep -r 'auth' src/"}   │
    │   │   └─ handle_bash_tool_response()         │
    │   ├─ bash {command: "cat src/main.py"}       │
    │   │   └─ handle_bash_tool_response()         │
    │   ├─ bash {restart: true}                    │
    │   │   └─ restart_bash_session()              │
    │   ├─ get_file_summary(...)                   │
    │   ├─ search_files(...)                       │
    │   └─ ... other tools                         │
    └─ Store section context in state              │
    │                                              │
    ▼                                              │
  document_section()  ◄── loops per section        │
    ├─ Author LLM writes section content           │
    ├─ Uses get_tech_spec_section tool only         │
    ├─ NO bash tool here                           │
    └─ Append content to updated_tech_spec         │
    │                                              │
    ▼ (loops back via document_router)             │
    │                                              │
    ├── [UPDATE mode] ─────────────────────────────┘
    │
    ▼
  summarize_changes()
    └─ process_section()
        ├─ Summarizer LLM (opus-4.5-thinking)
        ├─ LLM tool call loop:
        │   ├─ bash {command: "..."}
        │   │   └─ handle_bash_tool_response()
        │   ├─ bash {restart: true}
        │   │   └─ restart_bash_session()
        │   ├─ add_tech_spec_sub_section(...)
        │   ├─ mark_tech_spec_sub_section_complete(...)
        │   └─ ... other tools
        └─ Returns document_sections
    │
    ▼
  END
```

### Key Observations

1. **Single shared session** - `self.bash_session` lives on the helper instance,
   shared across all graph nodes throughout the entire pipeline run.

2. **Created once in setup** - After `download_repository_to_disk()`, the session
   is pointed at the downloaded repo directory.

3. **LLM controls restart** - The LLM decides when to restart via
   `{ "restart": true }` in tool args (e.g., if session hangs or gets corrupted).

4. **Two nodes use bash** - `gather_context` (search phase) and `process_section`
   (summarize phase). Both use the identical tool-handling pattern.

5. **No explicit cleanup** - The session is never explicitly `stop()`ed at the
   end of the pipeline. It relies on process cleanup when the job container exits.

6. **Bash tool is optional** - The LLM has bash available alongside other tools
   (file search, folder contents, web search). It chooses bash when it needs to
   run actual shell commands.

### Bash Tool Definition (from archie-shared)

The LLM sees this tool definition:
```python
ANTHROPIC_BASH_TOOL_NAME = "bash"
ANTHROPIC_BASH_TOOL_DEFINITION = {
    "name": "bash",
    "description": "Execute bash commands in the repository...",
    "input_schema": {
        "type": "object",
        "properties": {
            "command": {"type": "string", "description": "The bash command to execute"},
            "restart": {"type": "boolean", "description": "Restart the bash session"},
        },
    },
}
```

The LLM responds with tool calls like:
```json
{ "name": "bash", "args": { "command": "grep -rn 'class Auth' src/" } }
{ "name": "bash", "args": { "restart": true } }
```

---

## 4. BlitzyClient SDK - Current State

**File**: `blitzy-utils-python/blitzy_utils/blitzy_utils/blitzy_client.py`

### Available Methods

| Method | What It Does | Maps To |
|--------|-------------|---------|
| `get_client_url(company_id)` | Resolves mediator URL via admin API | N/A (internal) |
| `create_runner(company_id, job_id, ...)` | Creates K8s worker pod + Redis queue | `restart_bash_session()` first call |
| `delete_runner(company_id, job_id)` | Tears down worker pod | Missing in current code (no cleanup) |
| `create_job_command(company_id, job_id, command_text, ...)` | Submits command, returns `execution_id` | `handle_bash_tool_response()` (partial) |
| `get_command_status(company_id, command_id)` | Polls execution result | `handle_bash_tool_response()` (partial) |

### What Is Missing

| Missing | Why It's Needed |
|---------|----------------|
| `restart_session(company_id, job_id)` | LLM can request restart at any time. No method wraps `POST /api/v1/jobs/{id}/restart-session`. |
| `execute_command(company_id, job_id, command, ...)` | Submit + poll until terminal status. Without this, every consumer must write their own poll loop. |
| `execute_bash_tool(company_id, job_id, command, ...)` | Submit + poll + format output as `[stdout]\n\n...\n\n[stderr]\n\n...`. Direct drop-in for `handle_bash_tool_response`. |
| Async variants | All consumers are async (`await`). BlitzyClient uses sync `requests`. Need either async methods or `asyncio.to_thread()` wrapping. |
| URL caching | Every method calls `get_client_url()` which does an admin API round-trip. With dozens of commands per pipeline run, this adds significant latency. |

---

## 5. Integration Plan

### Phase 1: Extend BlitzyClient SDK (in `blitzy-utils-python`)

Add these methods to the `BlitzyClient` class:

```python
# 1. Restart session - wraps the mediator endpoint
def restart_session(self, company_id: str, job_id: str) -> dict:
    """
    Restart the worker's bash session.
    POST /api/v1/jobs/{job_id}/restart-session
    """
    server_url = self.get_client_url(company_id)
    url = f"{server_url.rstrip('/')}/api/v1/jobs/{job_id}/restart-session"
    response = self._http_client.post(url)
    response.raise_for_status()
    return response.json()

# 2. Execute and wait - submit + poll with backoff
def execute_command(
    self,
    company_id: str,
    job_id: str,
    command_text: str,
    timeout_seconds: int = 300,
    poll_interval: float = 1.0,
    max_poll_interval: float = 5.0,
) -> dict:
    """
    Submit command and poll until terminal status.
    Returns full status dict.
    """
    execution_id = self.create_job_command(
        company_id, job_id, command_text, timeout_seconds=timeout_seconds
    )
    interval = poll_interval
    while True:
        status = self.get_command_status(company_id, execution_id)
        if status.get("status") in ("completed", "failed", "timeout"):
            return status
        time.sleep(interval)
        interval = min(interval * 1.5, max_poll_interval)

# 3. Bash tool response formatter - matches handle_bash_tool_response output
def execute_bash_tool(
    self,
    company_id: str,
    job_id: str,
    command_text: str,
    timeout_seconds: int = 300,
) -> str:
    """
    Execute command and return LLM-formatted output string.
    Drop-in replacement for handle_bash_tool_response().
    """
    result = self.execute_command(
        company_id, job_id, command_text, timeout_seconds=timeout_seconds
    )
    stdout = result.get("stdout", "")
    stderr = result.get("stderr", "")
    output_parts = []
    if stdout:
        output_parts.append(f"[stdout]\n\n{stdout}")
    if stderr:
        output_parts.append(f"[stderr]\n\n{stderr}")
    return "\n\n".join(output_parts) if output_parts else "Command completed with no output."
```

Also add module-level convenience wrappers and matching unit tests.

### Phase 2: Update reverse-document-generator (and other consumers)

See Section 7 below for the exact diff.

---

## 6. SDK Gaps to Fill Before Integration

### P0 - Must Have (blocks integration)

| # | Gap | Action |
|---|-----|--------|
| 1 | **No `restart_session()` method** | Add `BlitzyClient.restart_session(company_id, job_id)` wrapping `POST /api/v1/jobs/{job_id}/restart-session` |
| 2 | **No `execute_command()` with polling** | Add `BlitzyClient.execute_command(company_id, job_id, command, ...)` that submits + polls until terminal status with backoff |
| 3 | **No `execute_bash_tool()` with output formatting** | Add `BlitzyClient.execute_bash_tool(company_id, job_id, command, ...)` that returns `[stdout]\n\n...\n\n[stderr]\n\n...` format matching `handle_bash_tool_response` |

### P1 - Should Have (quality/performance)

| # | Gap | Action |
|---|-----|--------|
| 4 | **Sync-only SDK, consumers are async** | Either add async variants using `httpx.AsyncClient`, or document that consumers should use `asyncio.to_thread(client.execute_bash_tool, ...)` to avoid blocking the event loop |
| 5 | **No URL caching** | Cache `get_client_url()` result per `company_id` with a TTL (e.g., 5 minutes). Currently every single SDK call triggers an admin API round-trip |
| 6 | **No runner cleanup pattern** | Add context manager support (`async with BlitzyClient.session(company_id, job_id) as session:`) or document explicit teardown pattern |

### P2 - Nice to Have

| # | Gap | Action |
|---|-----|--------|
| 7 | **`working_directory` / `environment_variables` in `create_job_command`** | These params are accepted by BlitzyClient but not forwarded by mediator's `CommandService`. Either remove from SDK or implement in mediator. |
| 8 | **Timeout mismatch docs** | Document that worker default is 300s but per-command `timeout_seconds` overrides it. Shared BashSession was 6000s. |

---

## 7. Consumer-Side Changes (reverse-document-generator)

### Changes to `__init__` (line 222)

```python
# BEFORE
self.bash_session = None

# AFTER
from blitzy_utils.blitzy_client import BlitzyClient
self.blitzy_client = BlitzyClient()
self.job_id = self.job_metadata.get("job_id") or self.job_metadata.get("blitzy_job_id")
```

### Changes to `setup()` (lines 351-366)

```python
# BEFORE
download_repository_to_disk(
    repo_name=self.repo_name,
    branch_name=self.branch_name,
    # ...
)

self.bash_session, _ = await restart_bash_session(
    bash_session=self.bash_session,
    repo_name=self.repo_name,
    branch_name=self.branch_name,
)

# AFTER
# download_repository_to_disk() call is REMOVED -
# the worker pod handles repo download using blitzy-utils + blitzy-client-utils.
# Worker calls main server (via HttpClient with API key) to get repo credentials,
# then clones the repo locally using download_repository_to_disk().

self.blitzy_client.create_runner(self.company_id, self.job_id)
```

**Note on repo download**: The worker will handle `download_repository_to_disk` itself.
The worker uses `client_utils.HttpClient` (from the shared `blitzy-client-utils` library)
to call the main server with an API key to fetch repo access tokens. The main server
acts as a facade — it internally calls the GitHub Handler service for SCM type detection
and access token retrieval, then returns the credentials to the worker. This means the
worker only needs an API key and git binary, with no GCP credential dependency.

If other graph nodes (like `gather_context` with `graph_builder`) also need local disk
access, the download may need to stay on the consumer side for those tools while bash
commands go remote.

### Changes to `gather_context()` bash tool handling (lines 450-469)

```python
# BEFORE
if restart:
    self.bash_session, _ = await restart_bash_session(
        bash_session=self.bash_session,
        repo_name=self.repo_name,
        branch_name=self.branch_name,
    )
    tool_result = "Bash session restarted successfully."
else:
    tool_result = await handle_bash_tool_response(
        command=command, session=self.bash_session
    )

# AFTER
if restart:
    self.blitzy_client.restart_session(self.company_id, self.job_id)
    tool_result = "Bash session restarted successfully."
else:
    tool_result = self.blitzy_client.execute_bash_tool(
        self.company_id, self.job_id, command
    )
```

If running in an async context and SDK is sync, wrap with:
```python
tool_result = await asyncio.to_thread(
    self.blitzy_client.execute_bash_tool,
    self.company_id, self.job_id, command
)
```

### Changes to `process_section()` bash tool handling (lines 544-563)

Identical change as `gather_context()` - same pattern, same replacement.

### Add cleanup (currently missing)

```python
# Add to a cleanup/teardown method or wrap pipeline execution:
async def run_pipeline(self, ...):
    try:
        # ... existing pipeline execution ...
    finally:
        self.blitzy_client.delete_runner(self.company_id, self.job_id)
```

### Imports to Change

```python
# REMOVE these imports:
- from blitzy_platform_shared.common.bash import handle_bash_tool_response, restart_bash_session

# ADD this import:
+ from blitzy_utils.blitzy_client import BlitzyClient
```

### Summary of Touched Lines

| Location | Lines | Change |
|----------|-------|--------|
| Imports | 27, 40-48 | Remove `handle_bash_tool_response`, `restart_bash_session`; add `BlitzyClient` |
| `__init__` | 222 | Replace `self.bash_session = None` with `BlitzyClient()` + `job_id` |
| `setup()` | 362-366 | Replace `restart_bash_session()` with `create_runner()` |
| `gather_context()` | 450-469 | Replace bash tool block with `execute_bash_tool()` / `restart_session()` |
| `process_section()` | 544-563 | Same replacement as `gather_context()` |
| New: cleanup | N/A | Add `delete_runner()` in finally block |
