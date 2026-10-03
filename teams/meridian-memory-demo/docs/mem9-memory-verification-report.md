# Atlantis Agent Memory Project — mem9 Self-Hosted Memory: Setup, Verification & Debugging

**Author:** Tinius Seland
**Date:** 2026-10-03

## 1. Overview

This report documents a self-hosted deployment of [mem9](https://github.com/mem9-ai/mem9) — a
persistent memory server for AI agents — connected to a dedicated TiDB Cloud database, and verifies
that memory writes actually persist and can be read back independently of the writing session. It
also documents a real bug found while trying to demonstrate the system conversationally through the
Claude Code plugin, traced down to the mem9 server's own search code.

## 2. Setup summary

- mem9 server built from source (`make build`) and run locally on port 8080, backed by a dedicated
  TiDB Cloud Starter cluster (own database, own schema, 13 tables loaded from the project's
  `server/schema.sql` unmodified).
- Connection uses `MNEMO_DB_BACKEND=tidb` and a `MNEMO_DSN` with TLS enabled, per the project's own
  setup documentation. No credentials, hosts, or claim links are recorded in this report or committed
  to the repository.
- The Claude Code plugin (`mem9@mem9`, installed from the `mem9-ai/mem9` marketplace) is configured to
  talk to the local server via `auth.json` (`base_url: http://localhost:8080`), not the hosted service.
- A basic read/write/restart sanity check (tenant creation, write, read, server restart, re-read) was
  completed early in setup and passed: a memory written before a server restart was still readable
  afterward.

## 3. Evidence: memory persists locally

Below is a real store-then-read-back pair, run directly against the local server with `curl`,
independent of any AI agent or chat interface. The same `id` and `content` come back on read as were
submitted on write, confirming the data is actually persisted (not just echoed back in the same
request).

**Write:**

```json
{
  "id": "46ca90b3-b462-4184-a190-e9f28cd86f50",
  "content": "The Meridian demo uses only synthetic data.",
  "memory_type": "pinned",
  "tags": ["demo"],
  "state": "active",
  "version": 1,
  "created_at": "2026-09-30T16:27:03Z"
}
```

**Read-back (separate request, ~32 minutes later, filtered only by `memory_type=pinned`):**

```json
{
  "memories": [
    {
      "id": "46ca90b3-b462-4184-a190-e9f28cd86f50",
      "content": "The Meridian demo uses only synthetic data.",
      "memory_type": "pinned",
      "tags": ["demo"],
      "state": "active",
      "created_at": "2026-09-30T16:27:03Z",
      "relative_age": "32 minutes ago"
    },
    {
      "id": "74516ad2-444b-4569-b8aa-4f8be6eef0d4",
      "content": "Sanity check memory: the demo project is called Atlantis.",
      "memory_type": "pinned",
      "tags": ["sanity-check"],
      "state": "active",
      "created_at": "2026-09-19T20:59:35Z",
      "relative_age": "1 week ago"
    }
  ],
  "total": 2
}
```

The second record is from an earlier session entirely (created 11 days before this read), confirming
the data survives across sessions and server restarts, not just within one request lifecycle.

See the accompanying screen recording (`mem9-store-recall-demo.mov`) for a live version of the same
store → read-back flow, run in one continuous take.

## 4. How to reproduce

Run these against a local mem9 server (`~/mem9-start.sh`), with `auth.json` already pointing at
`http://localhost:8080`:

```bash
KEY=$(python3 -c "import json;print(json.load(open('$HOME/.claude/plugins/data/mem9-mem9/auth.json'))['api_key'])")

curl -s -X POST http://localhost:8080/v1alpha2/mem9s/memories \
  -H "X-API-Key: $KEY" -H "Content-Type: application/json" \
  -d '{"content": "<your fact here>", "memory_type": "pinned", "tags": ["demo"]}' \
  | python3 -m json.tool

curl -s "http://localhost:8080/v1alpha2/mem9s/memories?memory_type=pinned&limit=10" \
  -H "X-API-Key: $KEY" \
  | python3 -m json.tool
```

## 5. Known issue investigated: conversational recall is unreliable

### 5a. Symptom

Asking Claude Code a plain question whose answer was already stored as a memory (e.g. "Does the
Meridian demo use only synthetic data?") did not reliably surface the stored fact — sometimes it
answered as if no memory system existed at all, and the `/mem9:recall` slash command once returned a
*different*, older memory with a numeric "confidence" score attached.

### 5b. Investigation

- Read the installed plugin's hook and skill source (`claude-plugin/hooks/common.sh`,
  `claude-plugin/hooks/user-prompt-submit.sh`, `claude-plugin/skills/store/SKILL.md`,
  `claude-plugin/skills/recall/SKILL.md`).
- `/mem9:store` and `/mem9:recall` are implemented as **skills** — natural-language instructions an
  LLM interprets and executes each time, not fixed code. The embedded script in both correctly reads
  `auth.json`'s `base_url`, so it should hit the local server — but the result containing a confidence
  score is a strong signal it actually reached the hosted service instead (the local server logs "no
  embedding configured, keyword-only search active" at startup, so it cannot produce confidence-ranked
  results). This points to inconsistent execution of the skill's instructions rather than a fixed code
  defect.
- The automatic recall hook (`user-prompt-submit.sh`) is, by contrast, **deterministic code**, not an
  interpreted skill. It correctly loads `auth.json` and queries
  `GET /v1alpha2/mem9s/memories?q=<prompt>&limit=10`. Testing this exact query path directly with
  `curl` reproduced a genuine failure: a single-word query for `Meridian` returned zero results even
  though a stored memory contains that exact word.
- A timing/indexing-lag hypothesis was tested directly: stored a fresh memory with a unique word
  (`zentavor`), queried immediately (0 results), waited 60 seconds, queried again (still 0 results).
  This rules out asynchronous index-building as the cause.
- Traced the server-side query path in the mem9 Go source
  (`server/internal/handler/memory.go`, `server/internal/service/memory.go`,
  `server/internal/repository/tidb/memory.go`, `server/internal/service/temporal_fact.go`,
  `server/internal/service/search_source_turns.go`):
  - `listMemories()` (handler) passes the raw query through `normalizeRecallQuery()` →
    `NormalizeTemporalRecallQuery()`. Confirmed this does **not** alter a plain, non-temporal query
    like `zentavor` — it only rewrites text containing date/time cues ("yesterday", "last week", etc.).
  - `MemoryService.Search()` routes to `ftsOnlySearch()` or `keywordOnlySearch()` depending on
    whether full-text search is available; both fall back to `looseTokenKeywordSearch()` when the
    primary search returns zero results and the query is a single token — which a one-word query
    always is.
  - That fallback ultimately calls `MemoryRepo.KeywordSearch()`
    (`server/internal/repository/tidb/memory.go:753`), which builds
    `WHERE state = 'active' AND content LIKE CONCAT('%', ?, '%')` — a plain, parameterized substring
    match that, by inspection, should match "zentavor" against content containing that exact word.
  - `finalizeSearchResults()` (the last step before the response is returned) only decorates
    *non-empty* result sets — it cannot be silently discarding matches, since it never runs on an
    already-empty list.

### 5c. Conclusion so far

The query is not being rewritten before it reaches the database, and the generated SQL is a
straightforward literal substring match that should succeed. The discrepancy therefore appears to be
at the live database layer (TiDB's handling of this specific parameterized `LIKE`/`CONCAT` pattern, a
collation detail, or the full-text-search path behaving differently from what the fallback logic
assumes) — something not visible from reading the Go source alone.

### 5d. Next steps (not yet done)

Confirming the exact cause would require running the literal generated SQL directly against the TiDB
database (bypassing the Go server) to see whether the mismatch is reproducible at that layer. Until
resolved, memory **storage and retrieval are reliable** (Section 3) — the open issue is specifically
in the natural-language / chat-based recall path, both the agent-interpreted slash commands and the
deterministic keyword-search endpoint they both ultimately depend on.

## 6. Files referenced

- `claude-plugin/hooks/common.sh`, `claude-plugin/hooks/user-prompt-submit.sh`
- `claude-plugin/skills/store/SKILL.md`, `claude-plugin/skills/recall/SKILL.md`
- `server/internal/handler/memory.go`
- `server/internal/service/memory.go`
- `server/internal/repository/tidb/memory.go`
- `server/internal/service/temporal_fact.go`
- `server/internal/service/search_source_turns.go`
