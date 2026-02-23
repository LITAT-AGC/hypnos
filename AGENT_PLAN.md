# AGENT_EXECUTION_PLAN

plan_version: 1
source_spec: DEFINITION.md
mode: machine_first
human_optimized: false

## 0. GLOBAL RULES

- Enforce strict project isolation by project root path hash for all stores.
- Use local/offline-first persistence only.
- Do not add out-of-scope features from Section 11 of DEFINITION.md.
- Keep synchronous methods synchronous when only using `better-sqlite3`.
- Ensure all SQLite connections run `PRAGMA journal_mode=WAL`.
- Implement explicit lifecycle close via `HypnosCore.close()` for all backends.

## 1. OUTPUT FILE CONTRACT

Required files to create/maintain:

- `src/database.js`
- `src/surreal-client.js`
- `src/vector-store.js`
- `src/event-listener.js`
- `src/context-builder.js`
- `src/hypnos-core.js`
- `src/mcp-server.js`
- `test/hypnos.test.js`
- `.gitignore`
- `.env.example`
- `package.json`

## 2. PHASED EXECUTION

### PHASE 1 — BOOTSTRAP

inputs:
- Node.js >= 18
- repo root

actions:
1. Initialize npm project in repo root.
2. Install dependencies:
   - `better-sqlite3`
   - `@modelcontextprotocol/sdk`
   - `surrealdb`
   - `pg`
   - `dotenv`
3. Create `.gitignore` with at minimum:
   - `node_modules/`
   - `.hypnos/`
   - `.env`
   - `*.db`
4. Create `.env.example` with all variables defined in Section 9.
5. Ensure package is ESM-compatible.
6. Add scripts:
   - `test`: `node --test test/hypnos.test.js`
   - `start`: run MCP server entrypoint.

exit_criteria:
- `npm install` completes.
- `npm run test` can execute test file path (even if tests are placeholders at this point).

### PHASE 2 — SQLITE FOUNDATION (`src/database.js`)

actions:
1. Implement `getProjectDb(projectRoot)`:
   - create `<projectRoot>/.hypnos/`
   - open/create `<projectRoot>/.hypnos/memory.db`
   - apply WAL mode
   - create `interaction_logs` schema idempotently
2. Implement `getGlobalDb()`:
   - resolve OS config directory (Windows `%APPDATA%`, Linux/macOS `~/.config`)
   - create `<config>/hypnos/`
   - open/create `global.db`
   - apply WAL mode
   - create `projects` schema idempotently
3. Surface meaningful error for unwritable/invalid paths.

exit_criteria:
- Both DB factories return usable `better-sqlite3` instances.
- Schema creation is idempotent.

### PHASE 3 — GRAPH STORE (`src/surreal-client.js`)

actions:
1. Implement `SurrealKnowledgeGraph` class.
2. In constructor:
   - hash `projectRoot`
   - derive project namespace `hypnos_<project_hash>`
   - read env for Surreal settings
3. Implement `connect()`:
   - connect/signin/use namespace+database
   - ensure required records/relations are available
4. Implement:
   - `upsertEntity(name, kind)`
   - `upsertRelation(sourceEntity, relation, targetEntity, sourceLogId)`
     - increment strength on repeated edge
     - append `sourceLogId`
   - `queryByEntity(entityName)`
   - `traversePath(fromEntity, toEntity, maxDepth=5)`
   - `close()`

exit_criteria:
- Repeated relation upserts reinforce edge strength.
- Query/traversal operations return structured graph data.

### PHASE 4 — VECTOR STORE (`src/vector-store.js`)

actions:
1. Implement `VectorStore` class.
2. In constructor:
   - hash `projectRoot`
   - derive schema `hypnos_<project_hash>`
   - load PostgreSQL env config
3. Implement `connect()`:
   - create pool
   - `CREATE EXTENSION IF NOT EXISTS vector`
   - create schema/table
   - create HNSW index on `embedding`
4. Implement:
   - `indexContent(sourceLogId, content, embedding)`
   - `search(queryEmbedding, nResults=5)` using cosine distance `<=>`
   - `close()`

exit_criteria:
- Embeddings insert and nearest-neighbor search return rows.

### PHASE 5 — CORE ORCHESTRATION (`src/hypnos-core.js`)

actions:
1. Implement `HypnosCore` constructor:
   - initialize project DB + global DB
   - upsert project registration in global DB
   - create graph/vector client instances (no remote connect yet)
2. Implement `init()` to connect graph and vector stores.
3. Implement synchronous:
   - `recordInteraction(type, content, feedback=0, metadata={})`
   - `getInteractions(filter?)`
4. Implement async:
   - `runSleepCycle()`
     - process logs where `user_feedback != 0`
     - for validated logs (`feedback=1`) extract triples and upsert graph
     - generate placeholder embedding and index to vector store
     - update `last_sleep_cycle`
     - return `{ status, processed, graphEntries, vectorized }`
   - `queryKnowledge(entity?)`
   - `traverseKnowledge(from, to, maxDepth?)`
   - `semanticSearch(queryEmbedding, nResults?)`
   - `close()` to close all DB/network resources

exit_criteria:
- Sleep cycle updates graph and vector stores and returns expected summary format.
- `close()` fully shuts down all active connections.

### PHASE 6 — PASSIVE EVENTS (`src/event-listener.js`)

actions:
1. Implement `EventListener` adapter taking `hypnosCore`.
2. Map callbacks to interaction writes:
   - `onFileChange` -> `code_fix`
   - `onSuggestionAccepted` -> `pattern`, feedback `1`
   - `onSuggestionRejected` -> `pattern`, feedback `-1`
   - `onTerminalOutput` -> `error` if exit != 0 else `suggestion`
   - `onConversationTurn` -> `preference`

exit_criteria:
- Each event method records expected `event_type`, `content`, `metadata`, and feedback.

### PHASE 7 — AUTO CONTEXT (`src/context-builder.js`)

actions:
1. Implement `ContextBuilder` with `buildContext(options?)` and `buildContextForFile(filePath)`.
2. Build output block:
   - `[Project Memory]`
   - `Recent Activity`
   - `Known Patterns`
   - `Relevant Context`
3. Pull data from:
   - recent interactions (default 20)
   - strongest graph relations
   - optional semantic results when `currentContext` present
4. Enforce max token budget (default 2000) with truncation.

exit_criteria:
- Output is deterministic, sectioned, and respects token cap.

### PHASE 8 — MCP SERVER (`src/mcp-server.js`)

actions:
1. Load `.env` via `dotenv`.
2. Resolve project root from init params or `HYPNOS_PROJECT_ROOT`.
3. Create and initialize `HypnosCore`.
4. Create `EventListener` + `ContextBuilder`.
5. Register resources:
   - `hypnos://memory/context`
   - `hypnos://memory/context/{filePath}`
6. Register tools:
   - `hypnos_search`
   - `hypnos_traverse`
   - `hypnos_sleep`
7. Register passive notification handlers:
   - `file_changed`
   - `suggestion_accepted`
   - `suggestion_rejected`
   - `terminal_output`
   - `conversation_turn`
8. Implement graceful shutdown and call `hypnosCore.close()`.

exit_criteria:
- Server initializes successfully and dispatches resources/tools/notifications to core components.

### PHASE 9 — TEST SUITE (`test/hypnos.test.js`)

Implement all required tests (1..18) from Section 8:

A. Core persistence:
1. Project isolation
2. Interaction recording + metadata round-trip
3. Feedback filtering

B. Passive events:
4. File change
5. Suggestion accepted
6. Suggestion rejected
7. Conversation turn

C. Sleep + knowledge/vector:
8. Sleep cycle consolidation
9. Relation strength reinforcement
10. Graph traversal
11. Semantic search

D. Context builder:
12. Context build sections
13. Context token limit
14. Context for file relevance

E. Lifecycle/errors:
15. Close lifecycle behavior
16. Invalid project root
17. Surreal connection failure
18. PG connection failure

test_runtime_rules:
- Use `node:test` + `node:assert`.
- Use temp dirs and clean in `t.after()`.
- Keep tests isolated and non-order-dependent.
- Integration tests require running SurrealDB and PostgreSQL/pgvector.

exit_criteria:
- `node --test test/hypnos.test.js` passes all 18 tests in provisioned environment.

### PHASE 10 — DOCS DELIVERY

actions:
1. Keep `DEFINITION.md` as canonical technical spec.
2. Add execution-oriented project docs (README and/or docs files) including:
   - setup prerequisites
   - `.env` configuration
   - start/test commands
   - architecture summary (passive write + automatic read + optional deep tools)
   - operational notes for SurrealDB/PostgreSQL services
3. Document shutdown and lifecycle guarantees (`close()`).

exit_criteria:
- A new agent can run setup, execute tests, and start MCP server from docs only.

## 3. VALIDATION SEQUENCE

Run in this order:
1. static module load check (all files import correctly)
2. SQLite-only tests
3. event/context tests
4. integration tests (Surreal + PGvector)
5. full suite

pass_condition:
- no failing tests
- no unclosed connection warnings
- required files present

## 4. NON-GOALS ENFORCEMENT

Do not implement now:
- fine-tuning pipeline
- scheduled sleep cycle automation
- cross-project federation
- embedding model/provider selection framework

## 5. FINAL ACCEPTANCE CONTRACT

Release is accepted when all conditions are true:
- Required file set exists.
- Public method signatures match DEFINITION.md.
- Synchronous/async boundary rules respected.
- WAL enabled for all SQLite DBs.
- Isolation verified for multi-project operation.
- MCP resources/tools/notifications functional.
- 18/18 required tests pass in configured environment.
