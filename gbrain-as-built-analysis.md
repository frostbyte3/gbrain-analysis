# GBrain — As-Built Code Analysis

> **What this is.** A reverse-engineering of [`garrytan/gbrain`](https://github.com/garrytan/gbrain) derived **only from source code**, with every prose document and every code comment removed first. It describes what the code *actually does*, grounded in `file:line` references, so it can be diffed against the intent/architecture doc (which was written from the project's own docs). Where the code's behavior is more specific, broader, or different from how such systems are usually described, that's called out in **§13 Notable as-built observations**.
>
> **Provenance.** Repo cloned at commit `814258d` (tag **v0.42.53.0**, 2026-06-24). Then:
> 1. Deleted all prose docs — 344 `*.md` files, `docs/`, `skills/` (128 files), `recipes/` (67 files), `llms.txt`/`llms-full.txt`.
> 2. Stripped **every comment** from all 1,974 `.ts`/`.tsx` files using the TypeScript compiler (parse → re-emit with `removeComments`), removing ~4.58 MB of comments with 0 parse errors. This is a true AST round-trip, so no code was altered — only comments and intent-narration are gone.
> The analysis below was then performed against that comment-free, doc-free tree.

---

## 1. What the code is (observed)

`gbrain` is a TypeScript CLI + MCP server, runtime **Bun ≥ 1.3.10** (`package.json` `engines`). It indexes markdown into Postgres-family storage and serves retrieval/synthesis to AI agents. The codebase is large: **739** source `.ts` files under `src/`, **143** CLI command modules, **1,199** test files, plus an `admin/` React SPA (8 `.tsx`).

### Actual dependency stack (`package.json`)
| Concern | Library |
|---|---|
| Embedded DB | `@electric-sql/pglite` 0.4.3 (Postgres-in-WASM) |
| Server DB driver | `postgres` 3.4 (porsager) + `pgvector` |
| LLM access | Vercel `ai` v6 + `@ai-sdk/anthropic`, `@ai-sdk/google`, `@ai-sdk/openai`, `@ai-sdk/openai-compatible`; also `@anthropic-ai/sdk`, `openai` |
| MCP | `@modelcontextprotocol/sdk` 1.29.0 |
| HTTP server | `express` 5 + `cors` + `cookie-parser` + `express-rate-limit` |
| Markdown | `gray-matter` (frontmatter), `js-yaml`, `marked` |
| **Code intelligence** | `web-tree-sitter` + `tree-sitter-wasms` |
| **Image/multimodal** | `@jsquash/png`, `@jsquash/avif`, `heic-decode`, `exifr` |
| Storage | `@aws-sdk/client-s3` |
| Misc | `@dqbd/tiktoken` (token counting), `zod` (validation), `chokidar` (file watch) |

Two facts the dependency list alone establishes that the prose summary underplays: **tree-sitter-based code parsing** and **image/multimodal ingestion** are first-class, shipped capabilities, not roadmap items.

---

## 2. Data model — the real schema (`src/schema.sql`, 1,425 lines)

**42 `CREATE TABLE` statements**, 91 indexes, **5 trigger functions / 5 triggers**, **34 tables with RLS enabled**, extensions `vector`, `pg_trgm`, `pgcrypto`.

### Core knowledge tables
- **`pages`** — `(source_id, slug)` unique (slugs are per-source). Columns include `page_kind CHECK IN ('markdown','code','image')`, `compiled_truth`, `timeline` (text), `frontmatter JSONB`, `content_hash`, `emotional_weight REAL`, `effective_date`, `last_retrieved_at`, `links_extracted_at`, and a `generation BIGINT` optimistic-concurrency counter.
- **`content_chunks`** — has **three** vector columns, not one: `embedding vector(1536)` (text), `embedding_image vector(1024)`, `embedding_multimodal vector(1024)`, plus `modality`. Also carries code-symbol fields (`symbol_name`, `symbol_type`, `start_line`, `end_line`, `parent_symbol_path TEXT[]`, `doc_comment`, `symbol_name_qualified`) and its own `search_vector TSVECTOR`.
- **`code_edges_chunk`** / **`code_edges_symbol`** — a dedicated **code graph** (caller/callee/symbol edges), separate from the knowledge graph.
- **`links`** — typed knowledge-graph edges with `link_type`, `link_source`, `link_kind`, `origin_page_id`, `resolution_type`; uniqueness is `UNIQUE NULLS NOT DISTINCT (from_page_id, to_page_id, link_type, link_source, origin_page_id)`.
- `tags`, `timeline_entries`, `raw_data` (API sidecars), `page_versions`, `files`, `ingest_log`, `config`.

### Facts / takes / calibration (the "opinion" layer)
`take_proposals`, `take_grade_cache`, `take_nudge_log`, `calibration_profiles`, `think_ab_results`, `dream_verdicts`. (The `facts`/`takes` row tables themselves are created by migrations, not the base `schema.sql`.) Status enums are enforced in-DB, e.g. `take_proposals.status CHECK IN ('pending','accepted','rejected','superseded')`, grade verdicts `CHECK IN ('correct','incorrect','partial','unresolvable')`.

### Job queue (BullMQ-shaped, in Postgres)
`minion_jobs` is a full queue row: `status CHECK IN ('waiting','active','completed','failed','delayed','dead','cancelled','waiting-children','paused')`, `lock_token`, `lock_until`, `delay_until`, `parent_job_id`, `on_child_fail CHECK IN ('fail_parent','remove_dep','ignore','continue')`, backoff config, `timeout_at`, token accounting, plus ~8 CHECK constraints on attempt/stall invariants. Companions: `minion_inbox`, `minion_attachments` (BYTEA *or* `storage_uri`), `subagent_messages`, `subagent_tool_executions` (status `pending|complete|failed`), `subagent_rate_leases`.

### Auth / audit
`access_tokens` (bearer, scope array), `oauth_clients` (DCR fields + `bound_tools`, `bound_source_id`, `bound_slug_prefixes`, `budget_usd_per_day`, `federated_read[]`), `oauth_tokens`, `oauth_codes` (PKCE `code_challenge`), `mcp_request_log`.

### Triggers (actual behavior)
1. `update_chunk_search_vector` (`schema.sql:359`) — chunk tsvector = `setweight(doc_comment,'A') || setweight(symbol_name_qualified,'A') || setweight(chunk_text,'B')`.
2. `update_page_search_vector` (`:818`) — page tsvector = `title(A) || compiled_truth(B) || timeline(C) || timeline_text(C)`, where `timeline_text` is assembled from the page's timeline. **The page FTS vector is rebuilt from the `pages.timeline` text column**, not by joining `timeline_entries` live.
3. `bump_page_generation` / `bump_page_generation_clock` (`:163`, `:243`) — maintain the optimistic-concurrency `generation` counter and a global `page_generation_clock` (cache-invalidation signal). *Not mentioned in the prose docs.*
4. `notify_minion_job_change` (`:1357`) — `pg_notify` on `minion_jobs` status changes → the queue uses **LISTEN/NOTIFY** for wakeups.

Vector indexes: HNSW (`vector_cosine_ops`) on `embedding` and on `embedding_image`; `embedding_multimodal` has no HNSW index in base DDL. FTS via GIN on `search_vector`; fuzzy title via `GIN(title gin_trgm_ops)`.

---

## 3. Operation contract (`src/core/operations.ts`)

A single array of operation objects, parsed via AST: **91 operations**, each with `name`, `description`, `params`, `scope`, `handler`, and optionally `mutating`/`localOnly`.

- **Scopes:** 51 `read`, 14 `write`, 24 `admin`, 2 `sources_admin`.
- **`localOnly` (7):** `purge_deleted_pages`, `sync_brain`, `file_list`, `file_upload`, `file_url`, `get_recent_transcripts`, `code_traversal_cache_clear`. These are refused on thin clients.
- **`mutating` (26):** includes `think` (it writes — synthesis is a mutation, see §7).

This one array is the source for both CLI verbs and MCP tool defs (`src/mcp/tool-defs.ts` maps `params` → JSON Schema). So the public MCP surface is **91 tools**, not "30+".

---

## 4. Request dispatch & auth (traced)

### Shared dispatcher — `src/mcp/dispatch.ts`
`dispatchToolCall(engine, name, params, opts)`: finds the op, runs `validateParams` (type/required check only — `:82`), builds an `OperationContext`, calls `op.handler(ctx, params)`. **It does not check scope.** It also redacts MCP params for logging (`summarizeMcpParams`, `:37`) to declared-keys + bucketed byte size.

### Four ways in
| Surface | File | Auth | Scope enforced? | `ctx.remote` |
|---|---|---|---|---|
| **CLI (local)** | `src/cli.ts` | none (local engine) | no | `false` (`cli.ts:545`) |
| **CLI (thin client)** | `src/cli.ts:235` | OAuth to remote host | remote-side | routed over MCP |
| **stdio MCP** | `src/mcp/server.ts` | none (local pipe) | no | `true`, `takesHoldersAllowList:['world']` |
| **HTTP — lightweight** | `src/mcp/http-transport.ts` | Bearer vs `access_tokens` | **no** | `true` |
| **HTTP — full** | `src/commands/serve-http.ts` | OAuth 2.1 (MCP-SDK router) | **yes** | `true` |

There are **two distinct HTTP servers**:
- `src/mcp/http-transport.ts` (295 lines, Bun `serve`) — bearer token against `access_tokens`, rate-limits, dispatches **any tool to any authenticated token**. No scope gate.
- `src/commands/serve-http.ts` (1,457 lines, Express + `@modelcontextprotocol/sdk` `mcpAuthRouter` + `GBrainOAuthProvider`) — full OAuth 2.1 (DCR registration, `authorization_code`+PKCE, `client_credentials`), admin dashboard, agent-scoped tokens. **Scope is enforced here and only here**, at `serve-http.ts:1029`:
  ```ts
  const requiredScope = op.scope || "read";
  if (!hasScope(authInfo.scopes, requiredScope)) { /* insufficient_scope, logged */ }
  ```

### Thin-client routing
`isThinClient(cfg)` ⇔ `cfg.remote_mcp` present (`config.ts:97`). When thin, `op.localOnly` ops are refused via `refuseThinClient` (`cli.ts:236`), everything else is proxied to the remote `mcp_url` (`runThinClientRouted`). Read-scope ops get a 180 s wall-clock timeout; write ops do not (`cli.ts:266-279`).

### Provenance trust gate (anti-spoofing)
`put_page` accepts `source_kind`/`source_uri`/`ingested_via` on the wire but **overrides them with server stamps unless `ctx.remote === false`** (`operations.ts` `put_page` handler). A remote write-scope token cannot forge audit-trail provenance — `ctx.remote === false` (local CLI/autopilot/dream only) is the sole condition that admits client-supplied provenance.

---

## 5. Engine contract & implementations

`BrainEngine` (`src/core/engine.ts`) is an interface with **139 members** (not "~47"). Two classes implement it: `src/core/pglite-engine.ts` and `src/core/postgres-engine.ts`. The interface is the real API surface area — retrieval (`searchVector`, `searchKeyword`, `searchKeywordChunks`, `relationalFanout`, `getAdjacencyBoosts`, `getBacklinkCounts`), graph (`traverseGraph`, `traversePaths`, `addLinksBatch`), facts/takes (~30 methods), code graph (`addCodeEdges`, `getCallersOf`, `getCalleesOf`, `getEdgesByChunk`), aliases (`resolveAliases`, `setPageAliases`), eval capture, and raw escape hatches (`executeRaw`, `executeRawDirect`). CLI/MCP never branch on engine type — they call the contract.

---

## 6. Ingestion pipeline (observed)

### Chunking (`src/core/chunkers/`)
Three strategies, all funneling into the recursive base:
- **`recursive.ts`** (`MARKDOWN_CHUNKER_VERSION = 3`): default `chunkSize = 300` **words**; 5-level `DELIMITERS` hierarchy `["\n\n"] → ["\n"] → sentence-enders → clause-enders → []` with **CJK-aware** splitting (`countCJKAwareWords`, CJK sentence/clause delimiters). Overlap is **character-based**, `Math.min(500, floor(maxChars/10))` (`recursive.ts:47`) — i.e. capped at 500 chars, *not* a fixed "50 words." Strips takes/facts fences before chunking.
- **`semantic.ts`**: groups by embedding similarity, falls back to recursive when a group exceeds `chunkSize*1.5`.
- **`llm.ts`**: LLM-guided boundaries over `CANDIDATE_SIZE` windows, recursive fallback.
- Plus **`code.ts` / `edge-extractor.ts` / `symbol-resolver.ts` / `qualified-names.ts`** — tree-sitter code chunking that emits symbol chunks and code-graph edges.

### Embedding (`src/core/embedding.ts`)
`BATCH_SIZE = 100`; default `EMBEDDING_MODEL = "text-embedding-3-large"`, `EMBEDDING_DIMENSIONS = 1536`. The model string is resolved through a **gateway** (`gatewayGetModel()`) that supports `provider:model` form, so the provider is pluggable; OpenAI is the default fallback only.

### Auto-link (`src/core/link-extraction.ts`)
Zero-LLM edge extraction, but **6 regexes**, not three (`link-extraction.ts:15-19, 55`):
1. `ENTITY_REF_RE` — markdown `[label](dir/slug)` gated by `DIR_PATTERN`.
2. `WIKILINK_RE` — `[[dir/slug|alias]]`.
3. `QUALIFIED_WIKILINK_RE` — `[[brain:dir/slug]]` cross-brain form.
4. `WIKILINK_GENERIC_RE` — bare `[[name]]`.
5. `MARKDOWN_LABEL_WIKILINK_RE` — nested label form.
6. `CODE_REF_REGEX` — `src/lib/...ts:line` source references across ~30 file extensions, for code brains.
Extraction masks already-matched ranges to avoid double-counting, resolves slugs via a `SlugResolver`, and writes through `addLinksBatch`. Link-type inference is heuristic and LLM-free.

### Parse / write
`put_page` (`operations.ts:680`) parses frontmatter (gray-matter), separates `compiled_truth` from `timeline`, chunks, embeds, reconciles tags, and — when `auto_link`/`auto_timeline` are on — runs `runAutoLink` + timeline extraction. Subagent writes are namespace-confined to `wiki/agents/<subagentId>/…` (anchored slug check in the handler). `content_hash` short-circuits unchanged content.

---

## 7. Retrieval & synthesis

### Hybrid search (`src/core/search/`, 33 files)
Entry `hybridSearch` (`hybrid.ts:396`). `RRF_K = 60` (`hybrid.ts:22`). Fusion is `rrfFusionWeighted([...lists])` over keyword + vector + (relational, for relational intents) arms. Observed stage order:
1. **Mode resolve** (`mode.ts`): `conservative` / `balanced` / `tokenmax` bundles.
2. **Intent classify** (`query-intent.ts`, deterministic) → intent weights (`intent-weights.ts`).
3. **Arms:** `searchVector` (HNSW, best-per-page pool), `searchKeyword` (tsvector), `relationalRecall`/`relationalFanout` (typed-edge arm, relational queries only).
4. **RRF fusion** → optional **`cosineReScore`**.
5. **`runPostFusionStages`** (`hybrid.ts:183`) applies, gated by flags: **backlink boost**, **salience boost**, **recency boost** (decay map), **title boost**, **graph signals** (`graph-signals.ts`: adjacency / cross-source / session-demote), **alias-resolved boost**. (This is a *broader* boost set than the docs' "adjacency/title/alias.")
6. **Exact-match boost**, **two-pass anchor expansion** (`two-pass.ts` — expand around top anchors, hydrate chunks).
7. **Reranker** (`rerank.ts`) — calls `gateway.rerank()`; provider/model are config-driven (ZeroEntropy `zerank-2` is the documented default but the seam is generic). Disabled ⇒ no-op.
8. **Dedup** (`dedup.ts`), **token-budget** (`token-budget.ts`), **autocut** (`autocut.ts`), **evidence/create_safety stamping** (`evidence.ts`).
Source boosts (`source-boost.ts`) `DEFAULT_SOURCE_BOOSTS`: `originals/ 1.5`, `writing/ 1.4`, `concepts/ 1.3` … `daily/ 0.8`, `media/x/ 0.7`, `openclaw/chat/ 0.5`, `archive/ 0.5`, `extracts/ 0.3`; hard-excludes are a separate set merged from env + opts.

### `think` synthesis (`src/core/think/`)
Pipeline: **`gather.ts`** runs `hybridSearch` + takes, renders a pages block (600-char excerpts) → **`sanitize.ts`** scrubs prompt-injection patterns from takes/content (e.g. regex for "ignore prior instructions", `sanitize.ts:6`) → LLM call with `THINK_SYSTEM_PROMPT_BASE` ("gbrain's synthesis engine", `prompt.ts:9`) → **`cite-render.ts`** parses inline + structured citations with a fallback path. `think` is marked `mutating` because it can write back. The **prompt-injection sanitization layer is a real, code-level security control** the prose docs don't surface.

---

## 8. Dream cycle (`src/core/cycle.ts`)

`ALL_PHASES` is an array of **19 phases** (`cycle.ts:10`): `lint, backlinks, sync, synthesize, extract, extract_facts, extract_atoms, resolve_symbol_edges, patterns, synthesize_concepts, recompute_emotional_weight, consolidate, propose_takes, grade_takes, calibration_profile, conversation_facts_backfill, enrich_thin, skillopt, embed` (+ a report-only `orphans`). `runCycle` (`cycle.ts:775`) runs `opts.phases ?? ALL_PHASES`; acquires the `gbrain_cycle_locks` lock only if a selected phase is in `NEEDS_LOCK_PHASES` (`cycle.ts:62, 795`). Phases are tagged `source`/`global`/`mixed` (`PHASE_SCOPE`) to control per-source vs brain-wide fan-out. Several phases are opt-in/pack-gated (`extract_atoms`, `synthesize_concepts`, `conversation_facts_backfill`, `skillopt`).

The actual scheduler is **`src/commands/autopilot.ts`**, a long-running loop (default `--interval 300` s, `autopilot.ts:240`) that calls `runCycle` and **adapts the interval to brain health**: `score >= 90 ? interval*2 : …` (`:700`) — it backs off when the brain is healthy. `src/commands/dream.ts` is the one-shot variant for external cron. So "66 cron jobs" is the operator's external schedule; the built-in driver is this adaptive loop.

---

## 9. Minions job queue (`src/core/minions/`, 47 files)

**Reliable Postgres queue.** `MinionQueue.claim` (`queue.ts:385`) atomically leases a job with `UPDATE minion_jobs SET lock_token=$1, lock_until=now()+$2ms … FOR UPDATE SKIP LOCKED`. Stalled jobs are reclaimed when `lock_until < now()` (`queue.ts:352`). Parent/child support uses `status='waiting-children'` and `FOR UPDATE` on the parent (`:103,:176`). Wakeups ride the `minion_job_notify` LISTEN/NOTIFY trigger.

**Crash-safe subagents** (`handlers/subagent.ts`). Each assistant message is persisted to `subagent_messages` and each tool call to `subagent_tool_executions` with `status pending→complete|failed`. On resume the handler inspects pending tool uses, and — critically — **throws if a non-idempotent tool is pending on resume** (`subagent.ts:229-230`: "non-idempotent tool … pending on resume; cannot safely re-run") rather than risk a double side-effect; idempotent tools re-run. `prompt_too_long` → `UnrecoverableError` (no retry). This is the real "two-phase pending→done" durability the docs reference, with a concrete safety rule.

Handlers present: `subagent`, `shell` (+ `shell-validate`/`shell-redact`/`shell-audit`/`shell-inherit`), `embed-backfill`, `ingest-capture`, `contextual-reindex-per-chunk`, `subagent-aggregator`, supervisor variants. Supporting machinery: rate leases, backpressure/lease-pressure audits, niceness, quiet-hours, stagger, budget meter/tracker, worker registry, child-worker supervisor.

---

## 10. Schema packs (`src/core/schema-pack/`, ~40 files)

`resolveActivePackName` (`registry.ts:47`) returns a `source` tagged with exactly the **7-tier** precedence: `per-call | env | per-source-db | db-config | gbrain-yml | home-config | default` — matching the documented chain precisely. Packs thread through reads/writes via `extractable.ts`, `expert-types.ts`, `enrichable.ts`, `link-inference.ts`, `page-to-alias.ts`, `page-to-link.ts`. Mutation is gated and audited (`mutate.ts`, `mutate-audit.ts`, `op-trust-gate.ts`), with `pack-lock.ts` for atomic file locks and `redos-guard.ts` enforcing the ReDoS protection on pack regexes (`lint-rules.ts` warns on nested quantifiers). `schema_apply_mutations` is `admin` scope.

---

## 11. Security controls (as implemented)

- **RLS** enabled on 34 tables (`schema.sql`).
- **Scope gating** only in the OAuth server (`serve-http.ts:1029`); the lightweight bearer server and stdio/CLI do not gate.
- **OAuth 2.1** via `GBrainOAuthProvider` (`oauth-provider.ts`): DCR, PKCE (`oauth_codes.code_challenge`), `client_credentials` + `authorization_code`, scope filtering (`scope.ts` `hasScope`/`assertAllowedScopes`), per-client `bound_tools`/`bound_source_id`/`bound_slug_prefixes`/`budget_usd_per_day`.
- **Provenance anti-spoofing** trust gate on `put_page` (§4).
- **Prompt-injection sanitization** before LLM synthesis (`think/sanitize.ts`).
- **Private-fence stripping** before chunking (`stripFactsFence`/`stripTakesFence` in chunkers) and on remote `get_page`.
- **SSRF / URL safety** (`ssrf-validate.ts`, `url-safety.ts`, `url-redact.ts`), timing-safe comparisons (`timing-safe.ts`), secret redaction (`source-config-redact.ts`, shell-redact handler), `destructive-guard.ts`, `gitleaks` config in repo.

---

## 12. Repository shape (code only, after doc strip)

| Area | Files | Notes |
|---|---|---|
| `src/core/` | ~585 `.ts` | engines, search (33), minions (47), cycle, schema-pack (~40), facts/takes/calibration, code-intel, chunkers, think |
| `src/commands/` | 143 | one module per CLI verb incl. 25+ `eval-*`, `serve`/`serve-http`, `autopilot`/`dream`, migrations |
| `src/mcp/` | 5 | `server.ts` (stdio), `http-transport.ts` (bearer), `dispatch.ts`, `tool-defs.ts`, `rate-limit.ts` |
| `src/schema.sql` | 1 | 42 tables, 91 indexes, 5 triggers, 34 RLS |
| `admin/` | React/Vite SPA | operator dashboard (Dashboard, Agents, Calibration, JobsWatch, RequestLog, Login) |
| `test/` + `tests/` | 1,199 | unit + e2e; `system-of-record-invariant.test.ts` exists and exercises rebuild |

---

## 13. Notable as-built observations (for the intent-vs-code comparison)

These are points where the *code* is more specific, broader, or different than a typical prose description. They are the seams worth focusing the later doc-vs-code diff on.

1. **Operation count:** 91 ops in the contract (51 read / 14 write / 24 admin / 2 sources_admin), surfaced as 91 MCP tools — vs prose "30+."
2. **Engine surface:** `BrainEngine` is **139 methods**, vs prose "~47 operations."
3. **Two HTTP servers, one enforces scope.** The lightweight `mcp/http-transport.ts` authenticates but does **not** scope-gate; only `commands/serve-http.ts` does. A reader who assumes "the HTTP server enforces scopes" should know *which* server.
4. **`content_chunks` is multimodal** (text 1536 + image 1024 + multimodal 1024) and **carries code-symbol metadata**; there is a separate **code graph** (`code_edges_*`). Code intelligence (tree-sitter) and image ingestion (jsquash/heic/exifr) are shipped.
5. **Chunk overlap is char-based (≤500), not "50 words."** Chunker version 3; CJK-aware.
6. **Auto-link uses 6 regexes** (incl. cross-brain `[[brain:slug]]` and source-code refs), not three.
7. **Page FTS is rebuilt from `pages.timeline` text** in the trigger, plus assembled timeline text — the "spans pages + timeline_entries" claim is approximate.
8. **Optimistic-concurrency `generation` clock** (two triggers) and **LISTEN/NOTIFY** job wakeups exist in-DB but aren't in the prose model.
9. **`think` is a *mutating* op** with a **prompt-injection sanitizer** and a structured+inline citation parser with fallback — more machinery than "retrieve then compose."
10. **Retrieval has more boosts than documented** (backlink, salience, recency, title, graph-signals{adjacency,cross-source,session-demote}, alias, exact-match) and a **two-pass anchor expansion** step.
11. **Reranker/embedding are gateway-abstracted** — ZeroEntropy/OpenAI are defaults, not hard dependencies.
12. **The cycle is 19 phases** with `NEEDS_LOCK`/`PHASE_SCOPE` taxonomy; the scheduler is an **adaptive autopilot loop** (health-based backoff), not literal cron.
13. **Minions crash-safety has a concrete rule:** a non-idempotent tool pending on resume **aborts** rather than re-running.
14. **Facts/takes base tables are migration-created**, not in `schema.sql`; the base file ships the *proposal/grade/calibration* tables. Disaster-recovery rebuild depends on the migration layer, not the base DDL alone.

---

*Method note: all assertions above were read from the comment-free, doc-free source tree at v0.42.53.0. `file:line` references point into that tree. Counts (91 ops, 139 engine methods, 42 tables, 19 phases, RRF_K=60, boost set, 6 link regexes) were extracted programmatically via the TypeScript AST or `grep` over the stripped source, so they reflect code, not documentation.*
