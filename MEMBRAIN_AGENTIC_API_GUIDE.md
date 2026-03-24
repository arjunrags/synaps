# Membrain API: Guide for Agentic Applications

> **Start here** if you are integrating Membrain from code: LangChain, Vercel AI SDK, AutoGen, MCP-adjacent servers, or a custom agent. This is the main markdown reference for HTTP usage and tool design.

This document is **framework-agnostic**. It explains the Membrain HTTP API, how semantic memory behaves, and how to wrap the API in **tools** (function calling) so any agent runtime—LangChain, Vercel AI SDK, AutoGen, custom orchestrators—can build reliable, memory-aware assistants.

For interactive schema exploration, use the deployed service’s **OpenAPI** (`/docs` or `/redoc`) when available.

---

## 1. What you are integrating

Membrain is a **semantic memory service** backed by PostgreSQL and pgvector. It is not a generic document store:

- **Atomic notes**: Each memory is one discrete fact or observation (not whole chat transcripts).
- **Graph links**: Related memories are connected; the backend can run **Guardian** logic (LLM-assisted) to propose links and evolution when ingesting.
- **Unified search**: Vector search returns **memory nodes** and **relationship edges** (edges can carry natural-language “question” descriptions that are also embedded—relationships are first-class in retrieval, not just graph metadata).

**Implication for agents:** Design tools that **search before answering personal questions**, **write concise facts**, and optionally **inspect graph structure** (paths, hubs, neighborhoods) when debugging or explaining reasoning.

---

## 2. Authentication and tenancy

| Method | Header | Use case |
|--------|--------|----------|
| API key | `X-API-Key: <key>` | Server-to-server, MCP servers, integrations |
| Clerk JWT | `Authorization: Bearer <jwt>` | End-user sessions in apps that use Clerk |

All memory data is scoped to the authenticated **user / organization** resolved by the server. There is no `X-User-ID` bypass in production—always send a valid API key or Bearer token.

**Tool design:** Load the API key from environment or a secrets manager; never embed keys in prompts. Pass the same client identity for an entire user session so memories stay consistent.

---

## 3. Base URL and API version

- **Prefix:** `/api/v1`
- **Example:** `POST https://<host>/api/v1/memories/search`

Health checks (no auth implied in typical deployments—confirm for yours):

- `GET /health`
- `GET /ready`

### 3.1 Terminals: Windows, PowerShell, and copy-paste

The **HTTP API is identical on every OS** — only shell tutorials differ. Docs and quick-starts often use **bash/zsh** (`export VAR=...`, `\` at end of line for multi-line `curl`). That matches **macOS**, **Linux**, **Git Bash**, and **WSL** out of the box.

| Environment | Setting env vars | `curl` / line breaks |
|---------------|------------------|------------------------|
| bash / zsh (macOS, Linux, Git Bash, WSL) | `export MEMBRAIN_API_KEY="mb_live_..."` | `\` continues a line |
| **PowerShell** | `$env:MEMBRAIN_API_KEY="mb_live_..."` | End each line with backtick `` ` `` for continuation, or use **one long line**; `curl` is often `curl.exe` |
| **Command Prompt (cmd.exe)** | `set MEMBRAIN_API_KEY=mb_live_...` | Use a **single-line** `curl` (no `\`), or Git Bash |

**Practical tip:** Use **Postman**, **Insomnia**, or your language’s HTTP client if you want zero shell differences. For CLI parity with the docs, **Git Bash** or **WSL** on Windows is the path of least friction.

---

## 4. Endpoint reference

### 4.1 Create memory (async ingest)

**`POST /api/v1/memories`**

Request body:

| Field | Type | Notes |
|-------|------|--------|
| `content` | string | Required. Max length **20,000** characters (atomic notes; stay well under this in practice). |
| `tags` | string[] | Optional. Max **50** tags; each tag max **256** chars. |
| `category` | string | Optional. Max **256** chars. |

**Response:** `202 Accepted` with an **ingest job** descriptor:

- `job_id` — poll for completion
- `job_status` — typically `queued` or `processing` initially
- `status_url` — absolute URL for `GET` polling

**`GET /api/v1/memories/jobs/{job_id}`**

Poll until `status` is `completed` or `failed`.

- **`completed`:** `result` contains `memory_id`, `action` (`created` | `updated`), and optional full `memory` object (shape matches read APIs).
- **`failed`:** `error` has `code`, `message`, and `retryable`.

**Why async:** Creation/update may run embedding generation, Guardian steps, and quotas. Clients **must** poll (or your tool wrapper should poll internally and return only the final memory summary).

### 4.2 Search

**`POST /api/v1/memories/search`**

| Field | Type | Default | Notes |
|-------|------|---------|--------|
| `query` | string | required | Natural language; max **10,000** chars. Prefer full questions over one-word queries. |
| `k` | int | `5` | 1–100. Number of hits. |
| `keyword_filter` | string \| string[] | null | **PostgreSQL regex** (`~*`) against **each tag string** (see [§5](#5-keyword-filter-regex-scoping-the-power-user-feature)). String = one pattern; **array** = **AND** (every pattern must match some tag on the same memory). |
| `response_format` | `"raw"` \| `"interpreted"` \| `"both"` | `"raw"` | See below. |

**`response_format`:**

| Value | Behavior |
|-------|----------|
| `raw` | Graph-native list: `memory_node` entries (with `related_memories` neighbors) and `relationship_edge` entries. Best for inspection, citations, and custom reasoning. |
| `interpreted` | Server-side LLM summarizes retrieval into `answer_summary`, `key_facts`, `important_relationships`, `conflicts_or_uncertainties`, supporting IDs, `confidence`. Results list may be empty when only synthesis is returned. |
| `both` | Interpreted block plus raw evidence (useful for debugging or showing evidence alongside a summary). |

If interpreted synthesis fails, responses may include `interpreted_error`; clients can fall back to `raw`.

**Search result shapes (raw):**

- **`type`: `memory_node`** — `id`, `content`, `tags`, `context`, `keywords`, `semantic_score`, **`related_memories`** (short previews of linked nodes—use these for “subconscious” context).
- **`type`: `relationship_edge`** — `description`, `source`, `target`, `score` — treat the edge as first-class evidence.

### 4.3 Read

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/api/v1/memories/{memory_id}` | Single memory (full detail). |
| `POST` | `/api/v1/memories/batch` | Body: `{ "memory_ids": ["...", "..."] }` — batch read. |

### 4.4 Update

**`PUT /api/v1/memories/{memory_id}`**

Body: at least one of `content`, `tags`. Used when the agent **knows the id** and needs a direct correction (user asked to fix a stored fact).

### 4.5 Delete

| Method | Path | Purpose |
|--------|------|--------|
| `DELETE` | `/api/v1/memories/{memory_id}` | Delete one memory. |
| `DELETE` | `/api/v1/memories/bulk` | Query: optional `memory_id` (takes precedence), or `tags`, `category` for filtered bulk delete. |

### 4.6 Unlink

**`POST /api/v1/memories/unlink`**

Body: `{ "memory_id_1", "memory_id_2" }` — removes the bidirectional link. Rare; use when the graph connection is wrong.

### 4.7 Stats and counts

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/api/v1/stats` | Totals, link density, **top_tags** (pairs of tag name and count). |
| `GET` | `/api/v1/memories/count` | Query: optional comma-separated `tags`, `category`. |

### 4.8 Graph analytics

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/api/v1/graph/path?from_id=&to_id=` | Shortest path between two memories (BFS over links). |
| `GET` | `/api/v1/graph/hubs?limit=` | Highest-degree nodes (default `limit=10`, max 100). |
| `GET` | `/api/v1/graph/neighborhood?memory_id=&hops=` | Nodes within `hops` (1–5). |
| `GET` | `/api/v1/graph/export` | Nodes + edges (+ link descriptions) for visualization. |

---

## 5. Keyword filter: regex scoping (the “power user” feature)

Semantic search alone answers “what is similar to this question?” **`keyword_filter`** answers “only among memories that **belong to this slice** of my tag namespace”—projects, bots, **time windows**, environments—without a separate database.

Implementation detail (so you design tags intentionally): the API applies **case-insensitive POSIX regex** to **each tag** in the memory’s `tags` JSON array. A memory passes if, for **every** pattern in the filter, **at least one** of its tags matches that pattern.

### 5.1 String vs array: OR inside, AND across patterns

- **Single string** — One regex. A tag matches if it matches the pattern. Use `|` for **alternation** inside that regex.

  Example: `"scope\\.project|scope\\.user"` matches memories that have **either** a tag like `scope.project` or `scope.user`—one pattern, alternation inside the regex.

- **Array of strings** — **Conjunction:** each pattern must match **some** tag on the **same** memory. Use this to combine dimensions (tenant + type + time bucket).

  Example: `["^container\\.mybot$", "type\\.preference|type\\.decision"]` means: must be in this container **and** must be typed as preference or decision. A typical “profile” flow runs parallel searches: e.g. `["scope\\.user", "type\\.preference|type\\.decision"]` for stable identity-like facts, and `"scope\\.project|scope\\.user"` in a separate call for broader work context.

### 5.2 Anchors and escaping (namespaced tags)

Tags often look like `type.decision` or `container.prod_bot`. In regex, `.` means “any character”. To match a **literal dot**, escape it: `type\.decision`, `^container\.[^.]+$`.

A robust pattern for **multi-tenant or multi-bot** setups is to prepend an **anchored** container tag to every search: e.g. `^container\.<id>$` combined with any additional patterns your app passes. If you call the HTTP API **directly**, you supply those regexes yourself; if you use a thin SDK wrapper, it may merge a container anchor with user-supplied filters automatically.

### 5.3 Relationship edges and filters

When `keyword_filter` is set, **relationship_edge** results are only included if **both** endpoint memories satisfy **all** the same patterns (each pattern must match a tag on the source **and** the same pattern must match a tag on the target). So heavy filters can hide edges even when the vector query is strong—if you need edges across scopes, run a second search with a looser filter or `keyword_filter: null`.

### 5.4 Temporal and time-sliced memories (tag design)

`keyword_filter` operates on **tags**, not on the memory row’s timestamp field (you can still sort or post-filter by `timestamp` in app code). For agent-controlled **“what happened this week / this sprint / in this session?”**, add **time or session tags at write time**, then filter with regex. If you only store a session id in app-side metadata, mirror it into a **`session.…`** tag when you need server-side regex scoping.

**Conventions you can adopt (examples):**

| Tag pattern | Meaning | Example `keyword_filter` |
|-------------|---------|---------------------------|
| `week.2026-W12` | ISO week bucket | `"week\\.2026-W12"` — this week only |
| `day.2026-03-23` | Calendar day | `"day\\.2026-03-23"` — one day |
| `month.2026-03` | Month bucket | `"month\\.2026-03"` |
| `session.<id>` | Chat / run id (sanitize to `[a-zA-Z0-9_-]`) | `"^session\\.abc123$"` or `"session\\..*2026"` for “sessions in 2026” |
| `era.phase2` | Product phase | `"era\\.phase2"` |
| `ttl.ephemeral` vs `ttl.permanent` | Retention policy | `"ttl\\.permanent"` to exclude throwaway captures |

**Combining time + scope:** use an array:

```json
"keyword_filter": ["^container\\.prod-bot$", "month\\.2026-03"]
```

That returns only March 2026 memories inside that container—still ranked by semantic similarity to `query`.

**Rolling windows:** regex can express ranges if your tag scheme is sortable (e.g. `day.2026-03-2[3-9]` for late March) or use a **single** pattern with alternation: `week\\.2026-W1[12]` for weeks 11–12. Prefer **simple buckets** (week/month) over fragile date regex unless you generate tags in code.

### 5.5 Recipes (copy-paste)

- **This calendar week only** — Tag on write: `week.<ISO>`; search: `"query": "What did we decide about auth?", "keyword_filter": "week\\.2026-W12"`.
- **User prefs vs project facts** — Tag `scope.user` / `scope.project`; filter static prefs: `["scope\\.user", "type\\.preference"]`.
- **One long-lived session thread** — Tag `session.<uuid>`; filter `"^session\\.550e8400"` (prefix) or full anchor.
- **Everything except ephemeral captures** — Tag `ttl.ephemeral` on throwaway rows; search with negative lookahead is **not** in SQL regex the same as PCRE—simpler to tag good rows `ttl.permanent` and filter that, or bulk-delete ephemerals by tag periodically.

### 5.6 Why this matters for agents

Without regex scoping, “recall what *we* decided” can pull similar memories from **other** projects or **older** eras. **`keyword_filter` + disciplined tags** give you deterministic slices; **`query`** still does the semantic ranking inside the slice.

---

## 6. Quotas and errors (agent-facing)

The API may enforce **plan tiers** (memory caps, weekly create quotas). Typical patterns:

- **`403`** with `TIER_MEMORY_CAP_REACHED` — stop creating until user upgrades or deletes data.
- **`429`** with `TIER_WEEKLY_QUOTA_REACHED` — backoff; surface `next_weekly_reset_at` if present.

**Tool design:** Map HTTP errors to short user-visible messages; for `429`, suggest retry after reset or reducing write frequency.

---

## 7. Building tools (framework-agnostic)

Below is a **logical tool set** you can map to your runtime’s function-calling or MCP-style tools. Name them to match your framework’s conventions.

### 7.1 Recommended tools

| Tool | Maps to | Responsibility |
|------|---------|----------------|
| `membrain_search` | `POST .../memories/search` | Primary read path; support `response_format` and `keyword_filter`. |
| `membrain_add` | `POST .../memories` + poll job | **Poll inside the tool** and return `memory_id` + `action` to the model. |
| `membrain_get` | `POST .../memories/batch` or `GET .../{id}` | Fetch full detail + evolution history when needed. |
| `membrain_update` | `PUT .../{id}` | Precise edits when ids are known. |
| `membrain_delete` | `DELETE` single or bulk | Prefer id-based delete when possible. |
| `membrain_unlink` | `POST .../unlink` | Optional; rare corrections. |
| `membrain_stats` | `GET .../stats` | Session introspection, debugging. |
| `membrain_graph_*` | path / hubs / neighborhood / export | Optional “analyst” or developer tools. |

### 7.2 Wrapper responsibilities

1. **Hide polling:** `membrain_add` should block (with timeout) until the job completes, then return a compact string or structured object your LLM can parse.
2. **Normalize errors:** Timeouts → “ingest still processing—retry”; `401` → configuration error, not user fault.
3. **Trim payloads:** For chat context, summarize raw search results (top N lines) unless the agent explicitly needs full JSON.
4. **Session scoping (optional):** Tag memories with something like `container.<id>` and always pass a matching `keyword_filter` so multiple sessions, bots, or tenants do not collide in retrieval.

### 7.3 Regex filters in your tool layer

Expose `keyword_filter` as an optional string or list of strings (regex). Document for your model when to use **project**, **container**, or **time-bucket** tags—see [§5](#5-keyword-filter-regex-scoping-the-power-user-feature). Helpers that **prepend** a container anchor (e.g. `^container\.myapp$`) reduce cross-tenant leakage.

### 7.4 Interpreted vs raw (when to expose to the model)

- **`interpreted`:** Fast answers for user questions (“What do we know about X?”) with less token use; good for consumer-facing assistants.
- **`raw`:** When the agent must cite **memory ids**, inspect **edges**, or verify **related_memories**.
- **`both`:** Audited answers (summary + evidence)—useful for trust-sensitive workflows.

---

## 8. Agent prompts and behaviors (copy-ready)

The following consolidates common agent instructions for memory-backed assistants. Adapt tone to your product.

### 8.1 Core system prompt (short)

You have persistent semantic memory (Membrain). **Synthesize** user requests with retrieved facts; do not merely list memories.

1. **Search before** answering questions that depend on the user’s history, preferences, or prior decisions.
2. Use **natural-language queries** (e.g. “What are the user’s constraints for choosing a laptop?”) not single keywords.
3. When search returns **`related_memories`**, use them to refine advice (e.g. dietary constraints next to food preferences).
4. **Store atomic facts** when the user shares stable information; skip greetings and ephemeral chit-chat.
5. Prefer **adding** new memories for new facets; use **update** when correcting a specific memory id; use **delete** when the user withdraws consent or data is wrong.

### 8.2 Expanded behaviors (for tool-heavy agents)

**Search**

- Before any personalized recommendation, call search.
- Read **`related_memories`** on each `memory_node` hit; they are deliberate graph expansions.
- For relationship questions, pay attention to **`relationship_edge`** hits—the edge text may answer better than a single node.

**Write**

- Store **third-person factual statements** (“User prefers…”, “Project X uses…”), not dialogue (“You said…”).
- Use **tags**: domain (`work`, `health`), type (`preference`, `constraint`, `goal`), priority (`important`, `temporary`)—stay consistent within your app.
- When the user changes a preference, either update the existing memory (if you hold its id from a recent search) or add a new memory with time context (“As of 2026-03, user prefers …”) if both old and new facts matter.

**Hygiene**

- Search before adding to reduce duplicate facts (many teams require “search first” before add).
- Use **stats** occasionally to notice empty or overloaded corpora.

### 8.3 Example tool descriptions (for your `tools` manifest)

Use variants like this in JSON Schema / OpenAI tool definitions:

**`membrain_search`**

> Semantic search over the user’s memory graph. Input: natural-language `query` (full sentences encouraged), optional `k`, optional `keyword_filter` (regex or list of regexes for tag AND-filtering), optional `response_format`: raw | interpreted | both. Returns memories, relationship edges, and/or an interpreted summary. Always call before answering personal or project-specific questions.

**`membrain_add`**

> Store a new atomic fact. Input: `content` (required), optional `tags`, optional `category`. Waits until ingest completes; returns memory id and whether it was created or updated. Search first if unsure whether the fact already exists.

---

## 9. Minimal HTTP examples

**Search (raw evidence)**

```http
POST /api/v1/memories/search
X-API-Key: YOUR_KEY
Content-Type: application/json

{
  "query": "What dietary restrictions should I consider when suggesting restaurants?",
  "k": 8,
  "response_format": "raw"
}
```

**Search with regex tag scoping (time bucket + container)**

```http
POST /api/v1/memories/search
X-API-Key: YOUR_KEY
Content-Type: application/json

{
  "query": "What did we decide about the rollout?",
  "k": 10,
  "keyword_filter": ["^container\\.prod-assistant$", "week\\.2026-W12"],
  "response_format": "both"
}
```

**Create (then poll)**

```http
POST /api/v1/memories
X-API-Key: YOUR_KEY
Content-Type: application/json

{
  "content": "Agreed to freeze API until Friday for QA sign-off",
  "tags": ["container.prod-assistant", "week.2026-W12", "scope.project", "type.decision"]
}
```

Then:

```http
GET /api/v1/memories/jobs/{job_id}
X-API-Key: YOUR_KEY
```

---

## 10. Checklist for shipping an agent

- [ ] API key or Clerk JWT wired securely; no keys in logs or prompts.
- [ ] Create path polls jobs until complete (or surfaces failure clearly).
- [ ] Search uses **questions**, not one-word queries, when possible.
- [ ] Tools expose **`response_format`** where users need summaries vs evidence.
- [ ] **Tag + `keyword_filter`** scheme for isolation (containers, projects, **time buckets**—see §5).
- [ ] Graph tools optional for power users or debugging.
- [ ] HTTP 403/429 from tier limits handled gracefully.
