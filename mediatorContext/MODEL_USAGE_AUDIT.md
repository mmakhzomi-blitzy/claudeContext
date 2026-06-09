# Model Usage Audit

**Purpose.** Inventory of LLMs and Anthropic-native features used across the `archie-job-*` repos and the shared library `archie-shared`, intended as the source-of-truth for the upcoming AWS conversation about **Anthropic 1P** (first-party).

**Scope.** 13 `archie-job-*` repos + `archie-shared` (the canonical LLM/tool definition library). Read from each repo's `origin/main` as of 2026-05-16.

**Not in scope.** Embeddings, prompt content, internal training/eval pipelines.

---

## 1. What "Anthropic 1P" means (one paragraph)

**1P (First-Party)** = the model vendor sells you the service directly. With Anthropic 1P you hit `api.anthropic.com` with an Anthropic API key, get full feature parity (Files API, computer use, web search/fetch tools, MCP, prompt caching, latest models), and Anthropic bills you. AWS now offers an arrangement where Anthropic 1P billing routes through your AWS account (typically via AWS Marketplace) so it counts toward AWS commitments and discounts — **without** moving you onto Bedrock (which lags on features and reshapes the API surface). This audit deliberately does **not** map our usage to Bedrock equivalents, because the direction we're considering is 1P-through-AWS, not Bedrock-through-AWS.

---

## 2. Canonical model definitions (archie-shared)

All `llm_*` constants in **current** `archie-shared/blitzy_platform_shared/common/llms.py` on `origin/main`. Consumer repos import these by name.

### Anthropic (LangChain `ChatAnthropic`)

| Constant | Model ID | max_tokens | Thinking | Notes |
|---|---|---|---|---|
| `llm_claude_opus_4_5_no_thinking` | `claude-opus-4-5-20251101` | 64000 | — | `anthropic-beta: computer-use-2025-11-24` |
| `llm_claude_opus_4_7_thinking_max` | `claude-opus-4-7` | 64000 | adaptive, summarized | `effort=max`, `service_tier=auto` |
| `llm_claude_opus_4_7_thinking_xhigh` | `claude-opus-4-7` | 64000 | adaptive, summarized | `effort=xhigh` |
| `llm_claude_opus_4_6_thinking_max` | `claude-opus-4-6` | 64000 | adaptive | `effort=max` |
| `llm_claude_opus_4_6_thinking_max_cx` | `claude-opus-4-6` | 64000 | adaptive | + `betas=[code-execution-web-tools-2026-02-09]` |
| `llm_claude_sonnet_4_6_adaptive` | `claude-sonnet-4-6` | 64000 | adaptive | `effort=max` |
| `llm_claude_sonnet_4_6_adaptive_cx` | `claude-sonnet-4-6` | 64000 | adaptive | + `betas=[code-execution-web-tools-2026-02-09]` |
| `llm_claude_sonnet_4_6_no_thinking` | `claude-sonnet-4-6` | 64000 | disabled | — |

### OpenAI (LangChain `ChatOpenAI`)

| Constant | Model ID | max_tokens | Reasoning |
|---|---|---|---|
| `llm_gpt5_4_mini` | `gpt-5.4-mini` | 64000 | `reasoning_effort=high`, Responses API |
| `llm_gpt5_4` | `gpt-5.4` | 64000 | `reasoning_effort=high` |
| `llm_gpt5_4_max` | `gpt-5.4` | 128000 | `reasoning_effort=high`, `verbosity=high` |
| `llm_gpt_5_5_high` | `gpt-5.5` | 64000 | `reasoning={effort:high, summary:auto}` |
| `llm_gpt_5_5_xhigh` | `gpt-5.5` | 64000 | `reasoning={effort:xhigh, summary:auto}` |

### Google + Other

| Constant | Model ID | Provider |
|---|---|---|
| `llm_gemini_3_pro` | `gemini-3-pro-preview` | Google (Vertex AI, location=global) |
| `llm_qwen3_5` | Qwen 3.5 (custom `ChatVertexQwen` wrapper) | Self-hosted on Vertex AI endpoint |

### Older constants still in use (via pinned `blitzy-platform-shared` versions)

The shared library went through a cleanup (commit `c9600ce`, 2026-01-29 "Cleanup old llms") removing many older constants. Several `archie-job-*` repos pin **older** versions of `blitzy-platform-shared` and still import these:

| Constant | Model ID | Used by (pinned version) |
|---|---|---|
| `llm_claude_4_sonnet_low_thinking_med_output` | `claude-sonnet-4-20250514` (thinking budget 16k, output 32k) | code-generator (0.0.1124), thinking-generator (0.0.1108) |
| `llm_claude_4_sonnet_med_thinking_max_output` | `claude-sonnet-4-20250514` (thinking budget 32k, output 64k) | repo-structure-generator (0.0.1124) |
| `llm_o3` | `o3-2025-04-16` | repo-structure-generator (0.0.1124) |
| `llm_gpt4_1` | `gpt-4.1-2025-04-14` | code-generator (0.0.1124), thinking-generator (0.0.1108) |
| `llm_claude_3_5_sonnet` | `claude-3-5-sonnet-20241022` (inline in code-validator) | code-validator |
| `llm_gpt4o` | `gpt-4o-2024-11-20` (inline) | code-validator |
| `llm_o1` | `o1-preview` (inline) | code-validator |

> **Risk callout for the AWS conversation:** model usage today spans **two generations** — repos on the latest shared (`0.0.1153–0.0.1155`) use Opus 4.7 + Sonnet 4.6 + GPT-5.x; older-pinned repos (`0.0.1108–0.0.1124`) still call Sonnet 4 (May 2025), GPT-4.1, o1, o3, GPT-4o. Any quote/commitment Anthropic provides should consider this drift — once the pinned repos update, Sonnet 4 usage moves to Sonnet 4.6 / Opus 4.7, shifting the volume between tiers.

---

## 3. Per-repo audit

13 `archie-job-*` repos. Pinned `blitzy-platform-shared` version is from each repo's `requirements.txt` on `origin/main`.

### LLM users (10 of 13)

| Repo | Pinned shared | Anthropic models | OpenAI models | Other |
|---|---|---|---|---|
| **code-generator** | 0.0.1124 | claude-sonnet-4-20250514 (`llm_claude_4_sonnet_low_thinking_med_output`) | gpt-4.1-2025-04-14 (`llm_gpt4_1`) | — |
| **code-graph-generator** | 0.0.1153 | claude-sonnet-4-6 (`llm_claude_sonnet_4_6_adaptive`) | gpt-5.4 (`llm_gpt5_4`), gpt-5.4-mini (`llm_gpt5_4_mini`) | — |
| **code-validator** | — (inline) | claude-3-5-sonnet-20241022 (inline) | gpt-4o-2024-11-20, o1-preview (inline) | — |
| **document-generator** | 0.0.1136 | claude-opus-4-7 (`llm_claude_opus_4_7_thinking_max`) | — | — |
| **repo-structure-generator** | 0.0.1124 | claude-sonnet-4-20250514 (`llm_claude_4_sonnet_med_thinking_max_output`) | o3-2025-04-16 (`llm_o3`) | — |
| **reverse-code-generator** | 0.0.1155 | claude-opus-4-7, claude-sonnet-4-6 | gpt-5.5 (`llm_gpt_5_5_xhigh`) | — |
| **reverse-document-generator** | 0.0.1153 | claude-opus-4-7 | — | — |
| **reverse-file-mapper** | 0.0.1153 | claude-opus-4-7 | gpt-5.5 | — |
| **reverse-thinking-generator** | 0.0.1153 | claude-opus-4-7 | gpt-5.5 | — |
| **thinking-generator** | 0.0.1108 | claude-sonnet-4-20250514 | gpt-4.1-2025-04-14 | — |

### Plumbing-only (3 of 13) — no LLM calls

- `archie-job-code-downloader` — repo download / clone orchestration
- `archie-job-code-uploader` — code commit / push orchestration
- `archie-job-tracker` — job state tracking

These don't appear in the Anthropic spend at all.

---

## 4. Anthropic-native features in use

These are 1P-only capabilities — they don't exist on Bedrock (or lag significantly). Their presence is the strongest argument for staying on Anthropic 1P rather than moving to Bedrock.

### Tool types (file-count per repo, production code only)

| Repo | web_search | web_fetch | bash | MCP | prompt_cache |
|---|---|---|---|---|---|
| code-generator | 1 | — | — | — | 1 |
| code-validator | — | — | — | — | 1 |
| document-generator | 1 | 1 | — | 1 | 1 |
| repo-structure-generator | — | — | — | — | 1 |
| reverse-code-generator | 2 | 1 | 2 | 1 | 1 |
| reverse-document-generator | 1 | 1 | 2 | 1 | 1 |
| reverse-file-mapper | 1 | 1 | 2 | 1 | 1 |
| reverse-thinking-generator | 2 | 1 | 1 | — | 1 |
| thinking-generator | — | — | — | — | 1 |

### Feature breakdown

**`web_search` tool** (`ANTHROPIC_WEB_SEARCH_TOOL_DEFINITION`):
- Tool types: `web_search_20250305` and `web_search_20260209` (Anthropic-native)
- Per-company allowlist/blocklist via `_apply_web_search_filter`
- **Used by 7 of 10 LLM repos.** Heavy usage.

**`web_fetch` tool** (`ANTHROPIC_WEB_FETCH_TOOL_DEFINITION`):
- Tool type: `web_fetch_20260209` (Anthropic-native)
- Same per-company filter mechanism
- **Used by 5 of 10 LLM repos.**

**Anthropic `bash` tool** (`ANTHROPIC_BASH_TOOL_DEFINITION` = `{type: "bash_20250124", name: "bash"}`):
- This is Anthropic's *computer-use* bash tool (different from our `BashSessionManager` execution model)
- **Used by 4 of 10 LLM repos** — all the reverse-* generators

**MCP (Model Context Protocol) servers**:
- `chrome-devtools` (Chrome CDP for browser automation)
- `figma` (via `get_figma_mcp`)
- `windows-mcp` (Windows computer-use, defined but reservation only)
- Used by document-generator + all reverse-* generators except reverse-thinking

**Prompt caching** (`cache_control: {type: "ephemeral"}`):
- **Used by 9 of 10 LLM repos.** Universal pattern.
- Anthropic-native; OpenAI has a different mechanism (automatic caching with no `cache_control` field).

### Anthropic betas in use

From the `llm_*` constants (in `archie-shared/common/llms.py`):

- `computer-use-2025-11-24` — on `llm_claude_opus_4_5_no_thinking`
- `code-execution-web-tools-2026-02-09` — on `llm_claude_opus_4_6_thinking_max_cx` and `llm_claude_sonnet_4_6_adaptive_cx`

From the older pinned versions (still active in production):

- `interleaved-thinking-2025-05-14` — on `llm_claude_4_sonnet_*` family (Sonnet 4)
- `computer-use-2025-01-24` — same
- `context-1m-2025-08-07` — same (1M context window)
- `prompt-caching-2024-07-31` — on `llm_claude_3_5_sonnet` (code-validator inline)

---

## 5. What this implies for an Anthropic 1P (on AWS) conversation

### Strong reasons to stay on Anthropic 1P (vs Bedrock)

1. **Web search / web fetch tools are 1P-only.** 7 of 10 LLM repos use `web_search`; 5 use `web_fetch`. These are Anthropic-native server tools — Bedrock doesn't expose them. A Bedrock move would require re-implementing search/fetch in our orchestration layer.
2. **Computer-use beta + MCP** — used by `document-generator` and the reverse-* generators. MCP server orchestration (Chrome DevTools, Figma) is built on top of Anthropic's tool-use semantics and the model needs to be aware of MCP context. Bedrock has been catching up but isn't at feature parity.
3. **Prompt caching** is universal in our code (9 of 10 LLM repos). Anthropic 1P caching is mature and well-priced. Bedrock has caching but with different semantics (and pricing tiers).
4. **Adaptive thinking** with named effort levels (`max`, `xhigh`, `high`) is part of Anthropic's native API. Bedrock exposes `thinking` but the configurability is more limited.
5. **`anthropic-beta` headers** are a 1P feature — Bedrock doesn't pass them through.

### Inventory facts to share with the AWS team

- **Active Anthropic models:** `claude-opus-4-7`, `claude-sonnet-4-6`, `claude-sonnet-4-20250514`, `claude-opus-4-5-20251101`, `claude-3-5-sonnet-20241022` (small footprint in `code-validator`).
- **Active OpenAI models:** `gpt-5.5`, `gpt-5.4`, `gpt-5.4-mini`, `gpt-4.1-2025-04-14`, `gpt-4o-2024-11-20`, `o1-preview`, `o3-2025-04-16`. Mix of Responses API + classic Chat Completions.
- **Active Google model:** `gemini-3-pro-preview` (Vertex AI, defined in shared but no consumer is currently importing it on `origin/main`).
- **Self-hosted on Vertex:** Qwen 3.5 (custom endpoint, defined but no consumer imports on `origin/main`).
- **All Anthropic calls go to `api.anthropic.com` today** — confirmed by `langchain_anthropic.ChatAnthropic` usage (no `langchain_aws.ChatBedrock` anywhere). No Bedrock dependencies exist yet.

### Migration-prep checklist

If we move billing to AWS Marketplace for Anthropic 1P:
- No code changes required — same API surface, same SDK, same model IDs.
- Same `anthropic-beta` headers continue to work.
- Same `web_search` / `web_fetch` / `bash` / MCP tool definitions work.
- Same prompt-caching behavior.
- Only billing destination changes.

---

## 6. Actual usage (from Anthropic + OpenAI dashboards)

Pulled from `Claude API Tokens Apr 2026.csv` (Anthropic Console export) and `OpenAI API Usage Apr to May 2026.csv` (OpenAI Dashboard export).

> **Window mismatch.** Anthropic CSV covers **Apr 1 – Apr 30, 2026 (30 days)**. OpenAI CSV covers **Apr 13 – May 13, 2026 (31 days)**. Not the 90-day window originally scoped — for a full 90-day picture, re-export both with a wider date range. Numbers below are exact for the windows covered.
>
> **Token counts are billable units.** Cache reads are billed at a fraction of regular input tokens (10% on Anthropic). Cache writes (5m/1h) are billed at a premium (1.25× / 2× of base input).

### 6.1 Anthropic — Apr 2026 (30 days)

**Total spend split by workspace** (output tokens as a rough volume proxy):

| Workspace | Input (no cache) | Cache write | Cache read | Output | Notes |
|---|---:|---:|---:|---:|---|
| **Blitzy Platform Prod** | 6.91 B | 25.27 B | 358.80 B | 4.46 B | Customer-facing product. **This is the audit's primary line.** |
| **Blitzy Internal** | 5.23 B | 4.67 B | 54.32 B | 1.08 B | Internal/staging environment + automated evals (the 4.1B haiku-no-cache is eval-loop traffic) |
| **Claude Code** | 0.02 B | 0.64 B | 8.11 B | 0.05 B | Anthropic's Claude Code dev tool — **not the archie product** |
| **Default** | 0.01 B | 0.00 B | 0.00 B | 0.00 B | Trivial test traffic |

#### Blitzy Platform Prod — per model (the line that matters for AWS sizing)

| Model | Input (no cache) | Cache write | Cache read | Output | Web search calls |
|---|---:|---:|---:|---:|---:|
| **claude-opus-4-7** | **5,277,296,282** | **21,265,498,268** | **300,003,221,033** | **3,729,584,225** | (see footnote) |
| claude-opus-4-6 | 1,298,193,446 | 3,873,147,413 | 56,473,075,533 | 683,179,284 | |
| claude-sonnet-4-6 | 188,748,602 | 133,918,220 | 2,325,563,678 | 39,002,228 | |
| claude-opus-4-5-20251101 | 100,870,626 | 1,769,691 | 0 | 11,820,039 | |
| claude-haiku-4-5-20251001 | 45,038,050 | 0 | 0 | 722,232 | |

Observations:
- **`claude-opus-4-7` is overwhelmingly dominant** — 5.3B input, 300B cache-read, 3.7B output. Roughly 5–10× any other model on this list.
- **`claude-opus-4-6` is the secondary heavy.** Likely the previous-gen workhorse before the Opus 4.7 rollout. Visible as the warm-tail of pinned-old consumer repos.
- **Sonnet 4.6, Opus 4.5, Haiku 4.5** are minor — used selectively (subagents, less-critical paths).
- **Cache-read tokens are ~57× input-no-cache tokens for Opus 4.7.** Prompt caching is doing massive heavy lifting. Anthropic 1P caching is mature; any Bedrock move that doesn't preserve caching parity would significantly inflate spend.

#### Blitzy Internal — per model (for completeness)

| Model | Input (no cache) | Cache write | Cache read | Output |
|---|---:|---:|---:|---:|
| claude-opus-4-7 | 818,823,345 | 3,085,146,748 | 34,311,844,996 | 632,455,854 |
| claude-opus-4-6 | 161,324,497 | 1,487,290,305 | 18,480,620,342 | 335,828,095 |
| **claude-haiku-4-5-20251001** | **4,138,781,624** | 5,746,311 | 185,120,040 | 91,032,802 |
| claude-sonnet-4-6 | 46,438,054 | 92,419,286 | 1,313,894,730 | 23,945,465 |
| claude-opus-4-5-20251101 | 48,341,525 | 36,329 | 290,478 | 1,014,536 |
| claude-nougat-eap | 507,751 | 2,856,959 | 27,481,238 | 297,279 |
| claude-sonnet-4-5-20250929 | 8,138,297 | 10,434 | 1,216,322 | 196,320 |
| claude-sonnet-4-20250514 | 7,069,468 | 96,001 | 2,094,517 | 169,615 |

The 4.1B haiku-no-cache traffic in Internal is the standout — almost certainly an eval/evaluation-loop pipeline (cheap model + no caching = batch evaluation pattern). Worth confirming with the team before quoting to AWS, since whether it migrates depends on the use case.

#### Anthropic-native feature usage (Prod + Internal, 30d)

- **Total web_search calls:** ~26.5k across Opus 4.7 + Opus 4.6 + Sonnet 4.6 — confirms heavy reliance on the 1P web_search tool that doesn't exist on Bedrock.
- **`claude-nougat-eap`** is an Anthropic early-access SKU used only in Internal. Worth flagging — EAP access patterns are 1P-only.

### 6.2 OpenAI — Apr 13 to May 13, 2026 (31 days)

**Total spend split by project:**

| Project ID | Requests | Input tokens | Cached | Uncached | Output | Notes |
|---|---:|---:|---:|---:|---:|---|
| `proj_tf2q4zTp4Er6Us9JRS8hWqoe` | 5,207,885 | 72.32 B | 15.91 B | 56.41 B | 27.15 B | **Production project — primary spend line** |
| `proj_GPJvxK7mTwNpRy35sF4dk2ZH` | 208 | 19.48 M | 17.18 M | 2.30 M | 124,711 | Trivial — internal/test |

#### Production project (`proj_tf2q4zTp4Er6Us9JRS8hWqoe`) — per model

| Model | Requests | Input total | Cached | Uncached | Output |
|---|---:|---:|---:|---:|---:|
| **gpt-5.4-mini-2026-03-17** | **5,158,312** | **66,038,168,879** | **15,348,399,872** | **50,689,769,007** | **26,258,924,374** |
| gpt-5.4-2026-03-05 | 49,423 | 6,269,749,142 | 552,841,344 | 5,716,907,798 | 892,711,274 |
| gpt-5.5-2026-04-23 | 150 | 15,495,593 | 8,978,560 | 6,517,033 | 304,435 |

Observations:
- **`gpt-5.4-mini` is the OpenAI workhorse** by an order of magnitude — 5.2M requests, 66B input, 26B output. Volume-wise comparable to (potentially higher than) the Anthropic Opus 4.7 workhorse, though Mini is much cheaper per token.
- **`gpt-5.4`** is the secondary high-quality option. ~50k requests, 6.3B input, ~890M output.
- **`gpt-5.5`** has tiny prod volume — probably brand new (release date in the model ID is 2026-04-23, late in the window). Adoption ramp likely starting after this CSV.
- **Older models in the static audit (`gpt-4.1`, `gpt-4o`, `o1`, `o3`)** show **zero** prod usage in this 31-day window despite being imported by pinned-old repos. Either those repos run very infrequently, or the imports are stale. Worth verifying before the AWS conversation — these repos may already be effectively decommissioned in prod traffic terms.

### 6.3 Summary — what the AWS team is sizing

If we project the 30-day Anthropic data to an annual estimate (rough — assumes steady-state, no growth):

**Anthropic (Blitzy Platform Prod, annualized):**
- claude-opus-4-7: ~63 B input + ~3.6 T cache-read + ~45 B output / year
- claude-opus-4-6: ~16 B input + ~680 B cache-read + ~8 B output / year
- Total Anthropic output: **~57 B output tokens/year** (mostly Opus tier)

**OpenAI (prod project, annualized from 31 days):**
- gpt-5.4-mini: ~780 B input + ~310 B output / year
- gpt-5.4: ~74 B input + ~10.5 B output / year
- Total OpenAI output: **~320 B output tokens/year** (Mini-dominant)

These are the headline numbers for the 1P contract conversation. AWS will likely want to see at least 90 days of data before committing — re-export both CSVs with `Jan 1 – Mar 31, 2026` and `Feb 13 – May 13, 2026` and rerun the aggregation to confirm.

### 6.4 Request and token rates (averages, limits, and headroom)

Useful for rate-limit conversations (AWS sells provisioned throughput in TPM / RPM, not monthly totals). Computed as the total over the CSV window divided by minutes in that window (30 days = 43,200 min for Anthropic; 31 days = 44,640 min for OpenAI).

#### Anthropic-allocated rate limits (current workspace ceilings)

These are what Anthropic has provisioned for the **Blitzy Platform Prod** workspace today.

| Metric | Limit |
|---|---:|
| Requests / minute (RPM) | **20,000** |
| Input tokens / minute (ITPM) | **18,000,000** |
| Output tokens / minute (OTPM) | **3,600,000** |

These are enterprise-tier limits. For comparison, our **observed averages over the 30-day window** sit *far* below:

| Metric | Limit | Current avg | Avg utilization | Peak factor needed to hit limit |
|---|---:|---:|---:|---:|
| RPM | 20,000 | not in CSV¹ | — | — |
| Input TPM (no cache + cache write + cache read combined) | 18,000,000 | ~9,000,000² | ~50% | ~2× |
| Output TPM | 3,600,000 | ~103,000 | ~2.9% | ~35× |

¹ Anthropic CSV doesn't include request counts.
² Anthropic counts cache-read tokens against ITPM at full rate even though they're billed at 10% — so the ITPM-relevant total is the **combined** stream, dominated by cache-read.

**Implication for AWS:** input is the constrained dimension at peak (because of the cache-read torrent), not output. Output has ~35× headroom; input has only ~2× headroom on average, less under peak. Any AWS-Anthropic 1P provisioning should match-or-exceed 18M ITPM with room for growth, not just 3.6M OTPM.

#### Anthropic — Blitzy Platform Prod (per-model averages)

| Model | Input TPM | Cache-read TPM | Output TPM |
|---|---:|---:|---:|
| **claude-opus-4-7** | **~120k** | **~7M** | **~85k** |
| claude-opus-4-6 | ~30k | ~1.3M | ~16k |
| claude-sonnet-4-6 | ~4k | ~55k | ~1k |
| claude-opus-4-5 | ~2k | — | — |
| claude-haiku-4-5 | ~1k | — | — |

**Total Anthropic Prod ≈ 9M TPM combined.** Cache-read is the dominant bucket (~8.3M TPM by itself).

> Anthropic CSV doesn't include request counts, so **Anthropic RPM isn't computable from this export.** Pull RPM separately from Anthropic Console if AWS asks.

#### OpenAI — Prod project (per-model averages)

| Model | RPM | Input TPM | Output TPM |
|---|---:|---:|---:|
| **gpt-5.4-mini** | **~115** | **~1.5M** | **~590k** |
| gpt-5.4 | ~1 | ~140k | ~20k |
| gpt-5.5 | ~0 | ~350 | tiny |

**Total OpenAI Prod ≈ 120 RPM, ~1.5M input TPM, ~600k output TPM.** gpt-5.4-mini is essentially the entire volume.

#### Headline rates for the AWS call

> **Anthropic prod: ~10M tokens/minute combined. Opus 4.7 is the workhorse — ~120k regular-input TPM, ~7M cache-read TPM, ~85k output TPM.**
>
> **OpenAI prod: ~120 RPM, ~2M tokens/minute total (1.5M input + 600k output). gpt-5.4-mini is essentially the whole load.**

#### Important caveat: averages vs peaks

These are **30/31-day averages**. Production traffic is bursty — peak TPM during job runs is typically **3–10× the average**. AWS provisions rate limits for peak, not average. To get peak rates:

1. Pull per-day breakdowns from Anthropic Console / OpenAI Dashboard, find the busiest single day, divide by 1440.
2. Or instrument LangSmith / Langfuse for per-minute timestamps.
3. Or quote the averages above with a **3–5× peak factor** as a defensible starting point until per-day data is available.

### 6.5 Where to verify / re-pull

| Source | What it gives you | How to access |
|---|---|---|
| **Anthropic Console** (`console.anthropic.com`) | Per-API-key token usage by model. CSV export. Used for §6.1. | Workspace owner login. |
| **OpenAI Dashboard** (`platform.openai.com/usage`) | Per-API-key token usage by model + per-day chart. CSV export. Used for §6.2. | Same — workspace owner login. |
| **Langfuse / LangSmith** | Trace-level data. Lets you compute per-feature (per archie-job-*) usage instead of just per-API-key. | See `LANGSMITH_PROJECT` env in each repo's `main.py`. |

### Tooling suggestions for ongoing tracking

If you want better-than-dashboard granularity:

1. **LangSmith filtered views** — group by model + project. You probably already have one project per `archie-job-*` (the env var pattern in their `main.py` files strongly suggests this). Build a dashboard view per project, sum tokens across the 90-day window.
2. **Anthropic Admin API** — for programmatic export of usage by API key / workspace.
3. **OpenAI Admin / Cost Management API** — programmatic equivalent for OpenAI side.
4. **Custom in-code metric** — add a thin `usage_metrics.py` callback that every `ChatAnthropic` / `ChatOpenAI` call sends to a Pub/Sub or directly to a BigQuery table. Useful if you want sub-per-job granularity (e.g., "Opus 4.7 in `gather_context` vs Opus 4.7 in `process_section`"). Not necessary for the AWS conversation, but worth doing eventually.
5. **OpenRouter-style cost-tracking middleware** — wrap the LangChain clients in a callback that computes cost from token counts using current pricing tables. Easier than reconciling spend after the fact.

---

## 7. Caveats and notes for the audit

1. **Working trees are on feature branches with uncommitted changes** in every audited repo. This report reads `origin/main` directly via `git show origin/main:<path>` and does not reflect anything in flight on those branches.
2. **archie-shared pinned-version drift.** 4 of 10 LLM repos pin shared versions older than the current cleanup (`0.0.1108`–`0.0.1124`). Their production model usage therefore differs from the constants visible in the shared repo's current main. Recommend bumping these as a separate hygiene task — relevant for the AWS conversation only if the older models (Sonnet 4 May 2025, GPT-4.1, o1-preview, o3) are still significant volume.
3. **`archie-job-code-validator`** is unusual — it defines its `ChatAnthropic` / `ChatOpenAI` clients **inline** in `main.py` rather than importing from shared. Older code style. Models used there (`claude-3-5-sonnet-20241022`, `gpt-4o-2024-11-20`, `o1-preview`) are the only places those specific model IDs appear in current main.
4. **Some `archie-shared` LLM constants are defined but not actively imported** by any `archie-job-*` on origin/main:
   - `llm_claude_opus_4_5_no_thinking`
   - `llm_claude_opus_4_7_thinking_xhigh`
   - `llm_claude_opus_4_6_thinking_max`, `llm_claude_opus_4_6_thinking_max_cx`
   - `llm_claude_sonnet_4_6_adaptive_cx`, `llm_claude_sonnet_4_6_no_thinking`
   - `llm_gpt5_4_max`, `llm_gpt_5_5_high`
   - `llm_gemini_3_pro`, `llm_qwen3_5`
   These may be used by repos outside the `archie-job-*` scope (e.g., backend services), or kept around for ad-hoc experiments.
5. **Test files excluded.** `*test*.py` and `main.test.py` files were excluded from the per-repo audit because some import constants that no longer exist in shared and would fail at runtime — those imports don't reflect actual production usage.

---

## 8. Quick FAQ for the AWS conversation

**Q: Are you on Bedrock today?**
A: No. All Anthropic calls go to `api.anthropic.com` via `langchain_anthropic.ChatAnthropic`. No `langchain_aws` or boto3-Bedrock imports anywhere.

**Q: What's the largest Anthropic SKU you use?**
A: `claude-opus-4-7` with adaptive thinking at `effort=max`, 64k output tokens, called by 5 archie-job-* repos including the heavy reverse-* generators. Likely the dominant Anthropic spend item.

**Q: What Anthropic-native features would block a Bedrock move?**
A: Web search, web fetch, computer-use bash, MCP orchestration, `anthropic-beta` headers, prompt-caching ergonomics. See Section 4.

**Q: What's the OpenAI vs Anthropic split?**
A: Both are used. Most jobs pair an Anthropic primary with an OpenAI fallback or secondary. OpenAI usage is significant — gpt-5.5 in 3 repos, gpt-5.4(-mini) in code-graph-generator, plus the older gpt-4.1 / gpt-4o / o1 / o3 in pinned-old repos. Full volume split should come from the dashboards (Section 6).

**Q: What's the total active model count?**
A: 5 Anthropic SKUs, 7 OpenAI SKUs, 1 Google SKU (defined, not actively imported), 1 self-hosted Qwen (defined, not actively imported). 14 model SKUs total defined in archie-shared.

---

## References

- **Canonical model definitions:** `archie-shared/blitzy_platform_shared/common/llms.py`
- **Tool definitions:** `archie-shared/blitzy_platform_shared/common/tools.py`, `code_generation/tools.py`
- **MCP server definitions:** `archie-shared/blitzy_platform_shared/mcp/consts.py`
- **Cache normalization logic:** `archie-shared/blitzy_platform_shared/common/utils.py` (`cache_control` handling near line 1201)
- **Recent shared cleanup:** commit `c9600ce` (2026-01-29 "Cleanup old llms")
