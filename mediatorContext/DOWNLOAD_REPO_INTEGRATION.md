# Download Repository Integration - Worker-Side Repo Cloning

> **Scope**: How `download_repository_to_disk` works today, what needs to change for the
> worker to handle repo cloning, and what APIs the main server needs to expose.
>
> **Companion docs**: See `ARCHITECTURE_REFERENCE.md` for system overview,
> `BASH_SESSION_INTEGRATION_ANALYSIS.md` for BashSession replacement details,
> `CODE_DOWNLOADER_ANALYSIS.md` for archie-job-code-downloader assessment.

---

## Table of Contents

1. [Current Flow: How download_repository_to_disk Works](#1-current-flow-how-download_repository_to_disk-works)
2. [Call Chain Deep Dive](#2-call-chain-deep-dive)
3. [Authentication: Two-Hop Token Flow](#3-authentication-two-hop-token-flow)
4. [The Problem: Why This Can't Run on the Worker As-Is](#4-the-problem-why-this-cant-run-on-the-worker-as-is)
5. [The Solution: Main Server as Credential Facade](#5-the-solution-main-server-as-credential-facade)
6. [APIs and Data: Main Server + EVENT_DATA](#6-apis-required-from-main-server)
7. [Worker-Side Implementation](#7-worker-side-implementation)
8. [Mediator Changes](#8-mediator-changes)
9. [File Index](#9-file-index)

---

## 1. Current Flow: How download_repository_to_disk Works

**Source**: `blitzy-utils-python/blitzy_utils/blitzy_utils/scm.py` (lines 205-319)

Today, `download_repository_to_disk` runs on **consumer pods** (reverse-document-generator,
reverse-code-generator, reverse-file-mapper). Each consumer:

1. Calls `download_repository_to_disk()` to clone the repo to local disk
2. Creates a `BashSession` pointed at the cloned directory
3. Runs commands against the repo via `BashSession`

```
Consumer Pod (e.g. reverse-document-generator)
├── download_repository_to_disk()   ← clones repo to /tmp/blitzy/{repo}/{branch}
├── restart_bash_session()           ← creates BashSession in cloned dir
└── handle_bash_tool_response()      ← runs commands via BashSession
```

In the new architecture, **the worker pod handles all of this**. The consumer only
talks to the mediator via `BlitzyClient` SDK.

---

## 2. Call Chain Deep Dive

```
download_repository_to_disk(repo_name, branch_name, user_id, server, commit_hash, git_project_repo_id)
│
├─ 1. _get_service_type(user_id, git_project_repo_id)
│     │  Detects SCM provider (GitHub / Azure DevOps / GitLab)
│     │  GET {SERVICE_URL_GITHUB}/v1/users/{user_id}/repositories/{git_project_repo_id}/svc-type
│     │  Auth: Google Cloud ID token (get_google_authorized_request_headers)
│     └─ Returns: (SvcType, hostname)
│
├─ 2. [Route by SCM type]
│     ├─ Azure DevOps → download_all_git_files_to_disk_azure_devops()  [separate flow]
│     ├─ GitLab       → download_all_git_files_to_disk_gitlab()        [separate flow]
│     └─ GitHub       → continues below (default)
│
├─ 3. get_git_repo_by_git_project_repo_id(git_project_repo_id)
│     │  Fetches access token for the repository
│     │  ServiceClient().get("github", "v1/github/repositories/{id}/access-token")
│     │  Auth: Google Cloud ID token (via ServiceClient)
│     └─ Returns: GitProjectRepo(access_token, repo_name, org_name, repo_id, ...)
│
├─ 4. get_cwd(repo_name, branch_name)
│     │  Resolves clone path on disk
│     └─ Returns: /tmp/blitzy/{repo_name}/{branch_name[:15]}
│
├─ 5. clone_repository_with_auth(full_repo_name, access_token, clone_path, branch_name, ...)
│     │  Clones repo using token-embedded HTTPS URL
│     │  Handles submodules with auth callback
│     └─ Returns: True/False
│
└─ 6. get_all_files_from_cloned_repo(clone_path)
      │  Walks directory tree
      └─ Returns: List[BlitzyGitFile]
```

### Function Signatures

```python
# scm.py
@blitzy_exponential_retry()
def download_repository_to_disk(
    repo_name: str,
    branch_name: str,
    user_id: str,
    server: str,
    commit_hash: str,
    git_project_repo_id: str,
    repo_id: Optional[str] = None,
    overwrite_existing_folder: bool = False,
    clean: bool = False,
) -> List[BlitzyGitFile]

# scm.py
@blitzy_exponential_retry()
def _get_service_type(user_id: str, git_project_repo_id: str) -> Tuple[SvcType, Optional[str]]

# scm_tools.py
def get_git_repo_by_git_project_repo_id(git_project_repo_id: str) -> GitProjectRepo

# github.py
def clone_repository_with_auth(
    full_repo_name: str,
    access_token: str,
    clone_path: str,
    branch_name: str = "main",
    commit_hash: Optional[str] = None,
    domain: Optional[str] = None,
    overwrite_existing_folder: bool = False,
    clean: bool = False,
) -> bool

# disk.py
def get_cwd(repo_name: str, branch_name: str) -> str
```

### GitProjectRepo Dataclass

```python
@dataclass
class GitProjectRepo:
    access_token: str        # OAuth/PAT token for SCM
    azure_org_id: str        # Azure org ID (or org name for other SCMs)
    azure_org_name: str      # Organization name
    azure_project_id: str    # Azure project ID (empty for non-Azure)
    repo_id: str             # Repository ID
    repo_name: str           # Repository name
    org_name: str            # Organization name
```

---

## 3. Authentication: Two-Hop Token Flow

Currently, credentials flow through **two hops**, both requiring GCP identity.
The GitHub Handler service lives in `archie-github-handler` (Flask app).

```
Consumer Pod                     archie-github-handler            SCM Provider
(GCP service account)            (Cloud Run)                      (GitHub/GitLab/Azure)
       │                                │                                │
       │  [Hop 1: GCP ID token]        │                                │
       ├───GET /v1/users/{uid}/         │                                │
       │   repositories/{id}/svc-type──►│                                │
       │◄──{svcType, hostname}──────────┤                                │
       │                                │                                │
       │  [Hop 1: GCP ID token]        │                                │
       ├───GET /v1/github/              │                                │
       │   repositories/{id}/           │                                │
       │   access-token────────────────►│                                │
       │                                ├──[Internal: fetch token]──────►│
       │◄──{access_token, org_name,     │◄──────────────────────────────┤
       │    repo_name, ...}─────────────┤                                │
       │                                                                 │
       │  [Hop 2: SCM access token]                                      │
       ├───git clone https://gho_...@github.com/org/repo.git───────────►│
       │◄──[repo contents]────────────────────────────────────────────  ┤
```

### archie-github-handler API Responses

**GET /v1/users/{user_id}/repositories/{git_project_repo_id}/svc-type**:
```json
{
  "svcType": "GITHUB",
  "hostname": null
}
```
- `svcType`: `GITHUB` | `AZURE_DEVOPS` | `GITLAB` | `GITLAB_SELF_HOSTED`
- `hostname`: `null` for GitHub/Azure, `"gitlab.com"` for GitLab cloud, custom for self-hosted

**GET /v1/github/repositories/{git_project_repo_id}/access-token**:
```json
{
  "access_token": "ghs_1234567890abcdef",
  "organization": "acme-corp",
  "installation_id": "12345678",
  "org_id": "O_xxxxx",
  "repo_id": "R_xxxxx",
  "repo_name": "my-app",
  "azure_org_id": null,
  "azure_project_id": null,
  "project_id": "391003c9-...",
  "id": "6f4e701e-..."
}
```

---

## 4. The Problem: Why This Can't Run on the Worker As-Is

| Requirement | Consumer Pod (today) | Worker Pod (K8s) |
|-------------|---------------------|-------------------|
| GCP service account | Has it (Cloud Run identity) | Does not have it |
| `ServiceClient` | Works (uses GCP ID tokens) | Cannot authenticate |
| `get_google_authorized_request_headers` | Works | Cannot authenticate |
| `git` binary | Available | Available |
| `blitzy-utils` library | Installed | Can be installed |
| Network to archie-github-handler | Yes (Cloud Run to Cloud Run) | Possible but auth fails |

The worker **can** run `git clone` and has access to `blitzy-utils`, but it **cannot**
call `archie-github-handler` directly because it lacks GCP credentials.

### Options Considered and Rejected

| Option | Why Rejected |
|--------|-------------|
| Give worker GCP credentials | Adds GCP dependency to K8s pods, complex credential rotation, security surface |
| Mediator fetches and passes credentials to worker | Passing tokens through Redis queue is a security concern |
| Mount GCP service account key in K8s | Manual key management, rotation burden, not scalable |

---

## 5. The Solution: Main Server as Credential Facade

The **main Blitzy server** mirrors the archie-github-handler APIs under `/v1/client/scm/`.
The worker calls these mirrored endpoints with a simple **API key** (`x-client-key`),
and the main server internally proxies to archie-github-handler. The existing `blitzy-utils`
library functions (`_get_service_type`, `get_git_repo_by_git_project_repo_id`) are reused
with a flag to route calls to the main server instead of archie-github-handler directly.

```
Worker Pod (K8s)                Main Blitzy Server        archie-github-handler    SCM Provider
(API key only)                  (proxies SCM APIs)        (Cloud Run)              (GitHub/GitLab/Azure)
       │                                │                        │                        │
       │  [API key: x-client-key]       │                        │                        │
       ├───GET /v1/client/scm/user/     │                        │                        │
       │   {uid}/repositories/          │  [GCP ID token]        │                        │
       │   {id}/svc_type───────────────►├───GET /v1/users/...───►│                        │
       │◄──{svcType, hostname}──────────┤◄───────────────────────┤                        │
       │                                │                        │                        │
       │  [API key: x-client-key]       │                        │                        │
       ├───GET /v1/client/scm/github/   │  [GCP ID token]        │                        │
       │   repositories/{id}/           │                        │                        │
       │   access-token────────────────►├───GET /v1/github/...──►│──[fetch token]────────►│
       │◄──{access_token, org_name,     │◄───────────────────────┤◄──────────────────────┤
       │    repo_name, ...}─────────────┤                        │                        │
       │                                                                                  │
       │  [access_token]                                                                  │
       ├───git clone https://gho_...@github.com/org/repo.git────────────────────────────►│
       │◄──[repo contents]──────────────────────────────────────────────────────────────┤
```

### Why This Approach

1. **No library changes** — existing `blitzy-utils` functions work as-is, just pointed at different base URL via flag
2. **No new credential types on worker** — only needs `API_KEY` and `MAIN_SERVER_URL` (already available via `blitzy-client-utils`)
3. **Single auth pattern** — worker uses `HttpClient` (API key) for everything, same as mediator
4. **Security** — access tokens never pass through Redis queues, fetched just-in-time by worker
5. **Decoupled** — worker doesn't know or care how the main server resolves credentials internally

---

## 6. APIs and Data: Main Server + EVENT_DATA

### 6.1 SCM APIs (Mirrored from archie-github-handler)

The main server mirrors the archie-github-handler APIs under `/v1/client/scm/`,
authenticated via `x-client-key` instead of GCP ID tokens. The existing `blitzy-utils`
library calls these same endpoints with a flag-based URL switch — no library changes needed.

**SCM Type Detection:**
```
GET /v1/client/scm/user/{user_id}/repositories/{git_project_repo_id}/svc_type
Headers:
  x-client-key: <api_key>

Response 200:
{
  "svcType": "GITHUB",
  "hostname": null
}
```

**GitHub Access Token:**
```
GET /v1/client/scm/github/repositories/{git_project_repo_id}/access-token
Headers:
  x-client-key: <api_key>

Response 200:
{
  "access_token": "ghs_1234567890abcdef",
  "organization": "acme-corp",
  "installation_id": "12345678",
  "org_id": "O_xxxxx",
  "repo_id": "R_xxxxx",
  "repo_name": "my-app",
  "azure_org_id": null,
  "azure_project_id": null,
  "project_id": "391003c9-...",
  "id": "6f4e701e-..."
}
```

**VCS-Agnostic Access Token (alternative):**
```
GET /v1/client/scm/repositories/{git_project_repo_id}/secret/access-token
Headers:
  x-client-key: <api_key>
```

These endpoints proxy to archie-github-handler internally. Response shapes
match the original archie-github-handler responses documented in Section 3.

### 6.2 EVENT_DATA Payload

The full job metadata payload is injected into the worker pod as a single env var
`EVENT_DATA` (JSON string). This serves as a **data source** — the worker extracts
only the specific fields it needs for each operation and constructs API calls
independently. `EVENT_DATA` is never passed as-is to any API.

```json
{
  "tech_spec_id": "46d00d62-a2eb-4824-b13c-8597c10740cf",
  "repo_name": "GitPracticeRepo",
  "repo_id": "71",
  "branch_id": "32c30f42-5dc7-4033-9967-12dabb2556af",
  "branch_name": "master-5",
  "company_id": "fc0ba5e0-48c1-401e-8556-9a3b99fbcf97",
  "user_id": "649beb9c-8657-4ecb-b68c-06a64f2c25a1",
  "team_id": "7dbd01a5-d412-472c-a706-54fa2bb7abfa",
  "job_id": "a4deb851-b482-4cb6-9de1-6c24d61e50ad",
  "project_id": "391003c9-8d6e-4ebe-95fb-6f0aca64816b",
  "head_commit_hash": "0e2d8e327b1af00b55e1538f671942408627341b",
  "prev_head_commit_hash": "0e2d8e327b1af00b55e1538f671942408627341b",
  "propagate": false,
  "git_project_repo_id": "6f4e701e-17b5-42d3-9256-2f686af07692",
  "job_type": "ADD_FEATURE",
  "previous_tech_spec_id": "0a9ca286-fa4a-4bca-ad40-738e7ae7808c",
  "document_mode": "UPDATE"
}
```

Fields the worker picks from `EVENT_DATA` per operation:

| Operation | Fields Used |
|-----------|------------|
| Repo credentials fetch | `git_project_repo_id`, `user_id` |
| Git clone | `repo_name`, `branch_name`, `head_commit_hash` |
| Job info lookup | `job_id` |
| Other main server calls | Cherry-picked per API — e.g. `company_id`, `project_id`, `tech_spec_id` as needed |

### 6.3 Worker → Main Server Communication Pattern

The worker uses `HttpClient` (from `blitzy-client-utils`) with API key authentication
to call main server APIs. `EVENT_DATA` provides the context (IDs, names, etc.) needed
to construct these calls.

```
Worker Pod
│
├── Startup
│   ├── Parse EVENT_DATA
│   ├── HttpClient (MAIN_SERVER_URL + API_KEY)
│   ├── download_repository_to_disk() via blitzy-utils (flag-based routing to main server)
│   │   ├── GET /v1/client/scm/user/{uid}/repositories/{id}/svc_type  → detect SCM type
│   │   ├── GET /v1/client/scm/github/repositories/{id}/access-token  → get credentials
│   │   └── git clone with credentials
│   └── Enter BLPOP loop
│
├── During Execution
│   ├── GET /v1/client/job/{job_id}                             → fetch job info / status
│   ├── POST /v1/client/environment/sync                        → trigger secret sync if needed
│   └── Any future APIs using EVENT_DATA fields
│
└── All calls use: x-client-key header (same pattern as mediator)
```

#### Main Server APIs (available to worker via HttpClient)

| Endpoint | Method | Purpose | EVENT_DATA Fields Used |
|----------|--------|---------|----------------------|
| `/v1/client/job/{blitzy_job_id}` | GET | Fetch job info | `job_id` |
| `/v1/client/environment/sync` | POST | Trigger Vault secret sync | — |
| `/v1/client/register` | PUT | Registration (mediator only) | — |
| `/v1/client/scm/user/{user_id}/repositories/{id}/svc_type` | GET | SCM type detection (proxies to archie-github-handler) | `git_project_repo_id`, `user_id` |
| `/v1/client/scm/github/repositories/{id}/access-token` | GET | GitHub access token (proxies to archie-github-handler) | `git_project_repo_id` |
| `/v1/client/scm/repositories/{id}/secret/access-token` | GET | VCS-agnostic access token (proxies to archie-github-handler) | `git_project_repo_id` |

The worker uses the existing `blitzy-utils` library functions with a flag to route
SCM calls through the main server instead of archie-github-handler directly.

---

## 7. Worker-Side Implementation

### 7.1 New Dependencies

Add to worker's `requirements.txt`:
```
blitzy-client-utils>=0.0.1   # HttpClient for main server communication
blitzy-utils>=0.0.38         # clone_repository_with_auth, get_cwd, etc.
```

### 7.2 Environment Variables

| Env Var | Source | Purpose |
|---------|--------|---------|
| `EVENT_DATA` | Full job metadata payload (JSON) | **Needs to be injected** — contains all job fields |
| `MAIN_SERVER_URL` | ConfigMap | Already injected (via env_from ConfigMap) |
| `API_KEY` | ConfigMap or env var | **Needs to be added** — for `HttpClient` auth |
| `JOB_ID` | Runner creation param | Already injected (kubernetes_service.py:135) |

`EVENT_DATA` replaces the need for individual env vars like `REPO_NAME`, `BRANCH_NAME`,
`USER_ID`, etc. The worker parses the JSON at startup and extracts whatever it needs.

### 7.3 Worker Startup Flow

The worker reuses the existing `blitzy-utils` library (`download_repository_to_disk`)
with a flag-based URL switch. When running on the worker, the library routes SCM API
calls to the main server (`/v1/client/scm/...`) instead of archie-github-handler directly.
No library code changes are needed — only the base URL configuration changes.

```python
import json
import os

from blitzy_utils.scm import download_repository_to_disk


def setup_repository():
    """Download repository at worker startup using existing blitzy-utils."""
    # Parse EVENT_DATA payload
    event_data = json.loads(os.getenv("EVENT_DATA", "{}"))

    # download_repository_to_disk uses the same SCM functions internally:
    #   _get_service_type() → GET /v1/client/scm/user/{uid}/repositories/{id}/svc_type
    #   get_git_repo_by_git_project_repo_id() → GET /v1/client/scm/github/repositories/{id}/access-token
    #
    # The flag-based routing points these calls at MAIN_SERVER_URL instead of
    # archie-github-handler, authenticated via x-client-key instead of GCP ID tokens.
    files = download_repository_to_disk(
        repo_name=event_data["repo_name"],
        branch_name=event_data["branch_name"],
        user_id=event_data["user_id"],
        server=event_data.get("server", ""),
        commit_hash=event_data["head_commit_hash"],
        git_project_repo_id=event_data["git_project_repo_id"],
        overwrite_existing_folder=True,
    )

    return files
```

The worker can also use other fields from `event_data` (e.g. `company_id`, `project_id`,
`job_type`) for any further communication with the main server.

---

## 8. Mediator Changes

### 8.1 Inject EVENT_DATA into Worker Pod

Update `kubernetes_service.py` `_build_container()` to inject the full job metadata
payload as a single `EVENT_DATA` env var (JSON string):

```python
import json

# job_info is already returned by expose_environments() → blitzy_service.get_job_info()
env_vars.append(client.V1EnvVar(
    name="EVENT_DATA",
    value=json.dumps(job_info),
))
```

This replaces the need for individual env vars. The existing `REPO_NAME` and
`BRANCH_NAME` injections (lines 146-147) can be kept for backward compatibility
or removed since they're redundant with `EVENT_DATA`.

### 8.2 Ensure API_KEY is Available to Worker

The `API_KEY` env var must be injected into worker pods. Options:
- Add to the common ConfigMap (`COMMON_CONFIGMAP_NAME`) — preferred, since `MAIN_SERVER_URL` is already there
- Inject as a direct env var in `_build_container()`
- Mount from a K8s Secret

---

## 9. File Index

### Current Implementation (blitzy-utils)

| File | Function | Purpose |
|------|----------|---------|
| `blitzy_utils/scm.py` | `download_repository_to_disk()` | Main entry — orchestrates SCM detection, credential fetch, clone |
| `blitzy_utils/scm.py` | `_get_service_type()` | Detects GitHub/Azure/GitLab via archie-github-handler |
| `blitzy_utils/scm_tools.py` | `get_git_repo_by_git_project_repo_id()` | Fetches access token via ServiceClient → archie-github-handler |
| `blitzy_utils/github.py` | `clone_repository_with_auth()` | Clones repo with token-embedded URL, handles submodules |
| `blitzy_utils/git_helpers.py` | `clone_repository_with_manual_submodules()` | Low-level git clone with manual submodule init |
| `blitzy_utils/git_helpers.py` | `get_all_files_from_cloned_repo()` | Walks cloned directory tree |
| `blitzy_utils/disk.py` | `get_cwd()` | Returns `/tmp/blitzy/{repo}/{branch[:15]}` |

### archie-github-handler (Cloud Run) — original APIs

| Endpoint | Purpose |
|----------|---------|
| `GET /v1/users/{user_id}/repositories/{id}/svc-type` | SCM type detection |
| `GET /v1/github/repositories/{id}/access-token` | Access token + repo metadata |
| `GET /v1/scm/repositories/{id}/secret/access-token` | VCS-agnostic access token (alternative) |

### Main Blitzy Server — mirrored SCM APIs (DONE)

| Endpoint | Proxies To | Purpose |
|----------|-----------|---------|
| `GET /v1/client/scm/user/{user_id}/repositories/{id}/svc_type` | archie-github-handler `/v1/users/.../svc-type` | SCM type detection |
| `GET /v1/client/scm/github/repositories/{id}/access-token` | archie-github-handler `/v1/github/.../access-token` | Access token + repo metadata |
| `GET /v1/client/scm/repositories/{id}/secret/access-token` | archie-github-handler `/v1/scm/.../access-token` | VCS-agnostic access token |

All mirrored endpoints use `x-client-key` header auth instead of GCP ID tokens.

### Integration Points (to modify)

| Component | Change |
|-----------|--------|
| **Main Blitzy Server** | Mirror archie-github-handler SCM APIs under `/v1/client/scm/` with API key auth (DONE) |
| **archie-client-mediator** `kubernetes_service.py` | Inject `EVENT_DATA` (full job metadata JSON) and ensure `API_KEY` is available to worker |
| **archie-client-worker** `worker.py` | Parse `EVENT_DATA`, add repo download at startup before BLPOP loop |
| **archie-client-worker** `requirements.txt` | Add `blitzy-client-utils`, `blitzy-utils` |
| **blitzy-utils** `scm.py`, `scm_tools.py` | Add env var flag to `_get_service_type()`, `get_git_repo_by_git_project_repo_id()`, and related functions — routes calls to main server SCM APIs or archie-github-handler based on environment |
