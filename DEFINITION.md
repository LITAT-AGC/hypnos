# Project Hypnos — Technical Specification

## 1. Purpose

Hypnos is a localized memory and "Generative Sleep" sidecar service. It enables AI coding agents (VS Code Copilot, Antigravity, OpenCode, or any MCP-compatible client) to persist knowledge, learned patterns, and user corrections across sessions. Its goal is to give agents long-term, project-scoped memory so they improve over time within a specific codebase.

### 1.1 The Forgetting Problem

LLMs routinely forget to call tools — especially low-priority ones like memory logging that don't directly solve the user's immediate task. The more tools exposed in a context window, the worse the recall rate. A model focused on fixing a bug will rarely stop to call `hypnos_record`.

Hypnos solves this with a **hybrid architecture** that removes the model from the critical path for both reading and writing memory:

| Concern | Mechanism | Model involvement |
|---|---|---|
| **Write** (capture interactions) | Passive event listener — hooks into IDE/client events | **None** (fully automatic) |
| **Read** (recall relevant memory) | MCP Resource — auto-injected into context | **None** (fully automatic) |
| **Deep query** (search, traversal) | MCP Tool — on-demand | **Optional** (model decides) |
| **Maintenance** (sleep cycle) | MCP Tool or scheduled | **Optional / admin** |

The common case (capturing and recalling knowledge) is fully automatic. MCP Tools are only used for explicit deep queries that the model actively chooses to run.

---

## 2. Core Principles

- **Project independence.** Each project maintains its own isolated memory store. Data from Project A must never leak into Project B. This is a hard architectural constraint.
- **Passive-first.** Memory capture and recall happen automatically. The model is never responsible for logging interactions or fetching context — that is handled by event listeners and MCP Resources.
- **Offline-first.** All persistence is local. No cloud dependencies for the core memory layer.
- **Lifecycle management.** Every database connection opened must be explicitly closeable. The core class must expose a `close()` method.
- **Transaction safety.** Relational writes must use PostgreSQL transactional guarantees and proper isolation levels where needed.
- **API honesty.** Public methods must only be marked `async` when they perform asynchronous operations.

---

## 3. Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| Runtime | Node.js 18+ | Native test runner, ESM support |
| Relational persistence | PostgreSQL (`pg`) | Project-scoped structured memory (interaction logs + metadata) |
| Graph persistence | SurrealDB | Knowledge graph storage with native graph traversals, relations, and path queries |
| Vector / semantic memory | PostgreSQL + `pgvector` | Embedding-based similarity search for knowledge retrieval |
| Agent protocol | `@modelcontextprotocol/sdk` | MCP server to expose Hypnos tools to agents |
| DB clients | `surrealdb`, `pg` | Node.js drivers for SurrealDB and PostgreSQL |
| Configuration | `dotenv` | Environment-specific settings |

All npm dependencies are installed in the project root. SurrealDB and PostgreSQL (with the `pgvector` extension) must be running as external services — see Section 9 for connection credentials.

---

## 4. Persistence Architecture

### 4.1 Project Relational Store — PostgreSQL

Each project is isolated in PostgreSQL using a project-specific schema (`hypnos_<project_hash>`). The schema stores interaction logs and any project-local relational metadata.

### 4.2 Global Metadata Store — PostgreSQL

Global metadata is stored in a dedicated shared schema (for example `hypnos_global`) in PostgreSQL. It tracks known projects, paths, and access/sleep-cycle timestamps. This schema must not store project-private knowledge content beyond metadata.

### 4.3 Knowledge Graph Store — SurrealDB

SurrealDB is used as a dedicated graph database for the knowledge graph. Each project gets its own **namespace and database** within SurrealDB, derived from a hash of the project root path, ensuring project isolation at the data layer. SurrealDB provides native graph traversals, relation records, and recursive path queries.

Connection credentials are read from the `.env` file (see Section 9).

### 4.4 Vector Store — PostgreSQL with pgvector

PostgreSQL with the `pgvector` extension is used for embedding-based semantic search. Each project gets its own table (or schema) within the database, namespaced by a project hash. This replaces ChromaDB and provides a single battle-tested RDBMS for both vector storage and standard SQL if needed.

Connection credentials are read from the `.env` file (see Section 9).

---

## 5. Database Schema

### 5.1 Project Relational Schema — PostgreSQL (`schema: hypnos_<project_hash>`)

**Table: `interaction_logs`** — Event-sourced record of every meaningful agent interaction.

| Column | Type | Description |
|---|---|---|
| `id` | BIGSERIAL, PK | Unique row ID |
| `timestamp` | TIMESTAMPTZ, DEFAULT NOW() | When the event occurred |
| `event_type` | TEXT | Category: `code_fix`, `preference`, `pattern`, `error`, `suggestion` |
| `content` | TEXT | The raw data, code snippet, or natural-language description |
| `user_feedback` | INTEGER | `1` = user validated/accepted, `-1` = user corrected/rejected, `0` = no feedback yet |
| `metadata` | JSONB | Structured context (file path, language, tags, etc.) |

### 5.2 Knowledge Graph — SurrealDB (`namespace: hypnos_<project_hash>`, `database: knowledge`)

SurrealDB uses a schema-less document/graph model. The following records and relations must be created:

**Record table: `entity`** — Nodes in the knowledge graph.

| Field | Type | Description |
|---|---|---|
| `id` | Record ID (auto) | SurrealDB record ID (e.g., `entity:useEffect`) |
| `name` | string | Human-readable name of the concept or symbol |
| `kind` | string | Category: `symbol`, `concept`, `file`, `pattern` |
| `created_at` | datetime | When this entity was first observed |

**Relation: `relates_to`** — Directed edges between entities.

| Field | Type | Description |
|---|---|---|
| `in` | Record ID | Source entity |
| `out` | Record ID | Target entity |
| `relation` | string | Relationship type (e.g., `causes`, `preferred_over`, `depends_on`) |
| `strength` | float, default 1.0 | Confidence weight; incremented when the same edge is reinforced |
| `source_log_ids` | array of int | References to `interaction_logs.id` entries that produced this edge |

Graph queries (traversals, shortest path, pattern matching) use SurrealDB's native `RELATE`, `->`, and `<->` syntax.

### 5.3 Vector Store — PostgreSQL + pgvector (`schema: hypnos_<project_hash>`)

**Table: `embeddings`** — Vectorized interaction content for semantic retrieval.

| Column | Type | Description |
|---|---|---|
| `id` | SERIAL, PK | Unique row ID |
| `source_log_id` | INTEGER | Reference to the originating `interaction_logs.id` |
| `content` | TEXT | Original text content |
| `embedding` | vector(1536) | Embedding vector (dimension depends on the chosen model; 1536 for OpenAI `text-embedding-3-small`, adjust as needed) |
| `created_at` | TIMESTAMPTZ, DEFAULT NOW() | When the embedding was created |

An **IVFFlat or HNSW index** must be created on the `embedding` column for efficient approximate nearest-neighbor search.

### 5.4 Global Metadata Schema — PostgreSQL (`schema: hypnos_global`)

**Table: `projects`**

| Column | Type | Description |
|---|---|---|
| `id` | BIGSERIAL, PK | Unique row ID |
| `path` | TEXT, UNIQUE | Absolute path to the project root |
| `last_accessed` | TIMESTAMPTZ | Last time Hypnos was active on this project |
| `last_sleep_cycle` | TIMESTAMPTZ, NULLABLE | Last time a sleep cycle completed for this project |

---

## 6. Module Structure

The project must produce the following files at minimum:

```
hypnos/
├── src/
│   ├── database.js          # PostgreSQL connection factory (project + global schemas)
│   ├── surreal-client.js    # SurrealDB connection & knowledge graph operations
│   ├── vector-store.js      # PGvector connection & embedding operations
│   ├── event-listener.js    # Passive event capture (write path)
│   ├── context-builder.js   # Builds auto-injected context summaries (read path)
│   ├── hypnos-core.js       # Core logic class (orchestrates all stores)
│   └── mcp-server.js        # MCP server: Resources (auto) + Tools (on-demand)
├── test/
│   └── hypnos.test.js       # Test suite (Node.js native test runner)
├── .gitignore
├── .env.example
├── package.json
└── DEFINITION.md
```

---

## 7. Module Responsibilities

### 7.1 `database.js` (PostgreSQL relational layer)

- Export a `getPostgresPool()` function that:
  1. Reads PostgreSQL credentials from environment variables.
  2. Creates and returns a shared `pg` connection pool.

- Export an `ensureProjectSchema(pool, projectRoot)` function that:
  1. Derives `hypnos_<project_hash>` from `projectRoot`.
  2. Creates the schema if it does not exist.
  3. Creates `interaction_logs` idempotently (`CREATE TABLE IF NOT EXISTS`) per Section 5.1.
  4. Returns the schema name.

- Export an `ensureGlobalSchema(pool)` function that:
  1. Creates `hypnos_global` schema if it does not exist.
  2. Creates the `projects` table idempotently per Section 5.4.

### 7.2 `surreal-client.js` (Knowledge Graph)

Export a `SurrealKnowledgeGraph` class:

- **`constructor(projectRoot)`** — Derives a project-specific namespace from a hash of `projectRoot`. Reads SurrealDB connection credentials from environment variables (Section 9).
- **`async connect()`** — Opens a connection to SurrealDB, selects the project namespace and database, and ensures the `entity` table and `relates_to` relation schema exist.
- **`async upsertEntity(name, kind)`** — Creates an entity record if it doesn't exist; returns the record ID.
- **`async upsertRelation(sourceEntity, relation, targetEntity, sourceLogId)`** — Creates or updates a `relates_to` edge between two entities. If the exact `source→relation→target` edge already exists, increments `strength` by 1.0 and appends the `sourceLogId` to the `source_log_ids` array.
- **`async queryByEntity(entityName)`** — Returns all outgoing and incoming relations for a given entity with their targets and strengths.
- **`async traversePath(fromEntity, toEntity, maxDepth?)`** — Uses SurrealDB graph traversal to find paths between two entities up to `maxDepth` (default 5).
- **`async close()`** — Closes the SurrealDB connection.

### 7.3 `vector-store.js` (Semantic Memory)

Export a `VectorStore` class:

- **`constructor(projectRoot)`** — Derives a project-specific schema name from a hash of `projectRoot`. Reads PostgreSQL connection credentials from environment variables (Section 9).
- **`async connect()`** — Opens a connection pool to PostgreSQL, creates the project schema if it doesn't exist, ensures the `pgvector` extension is enabled (`CREATE EXTENSION IF NOT EXISTS vector`), and creates the `embeddings` table with an HNSW index on the `embedding` column.
- **`async indexContent(sourceLogId, content, embedding)`** — Inserts a row into the `embeddings` table with the provided embedding vector.
- **`async search(queryEmbedding, nResults?)`** — Performs a nearest-neighbor search using cosine distance (`<=>`) against the `embedding` column. Returns the top `nResults` (default 5) rows with their content and distance.
- **`async close()`** — Closes the PostgreSQL connection pool.

**Note:** Embedding generation (converting text to vectors) is **not** the responsibility of `vector-store.js`. The caller (`hypnos-core.js`) must provide pre-computed embeddings. The embedding model and provider are out of scope for the initial build (see Section 11).

### 7.4 `event-listener.js` (Passive Write Path)

Export an `EventListener` class that captures interactions **without any model involvement**:

- **`constructor(hypnosCore)`** — Takes a reference to an initialized `HypnosCore` instance for recording events.
- **`onFileChange(filePath, diff)`** — Called when a file is saved. Records a `code_fix` interaction with the diff content and file path in metadata.
- **`onSuggestionAccepted(suggestion, filePath)`** — Called when the user accepts an AI suggestion. Records a `pattern` interaction with `user_feedback = 1`.
- **`onSuggestionRejected(suggestion, filePath)`** — Called when the user rejects or corrects an AI suggestion. Records a `pattern` interaction with `user_feedback = -1`.
- **`onTerminalOutput(command, output, exitCode)`** — Called after a terminal command completes. Records an `error` interaction if the exit code is non-zero, or a `suggestion` if it succeeds.
- **`onConversationTurn(role, content)`** — Called on each user/assistant message in a chat session. Records a `preference` interaction.

The implementing client (VS Code extension, OpenCode hook, etc.) is responsible for calling these methods at the right time. `event-listener.js` does not hook into any specific IDE — it is a protocol-agnostic adapter.

**Key principle:** The LLM never calls these methods. They are driven by the client/IDE runtime.

### 7.5 `context-builder.js` (Automatic Read Path)

Export a `ContextBuilder` class that constructs a memory summary for auto-injection into the model's context:

- **`constructor(hypnosCore)`** — Takes a reference to an initialized `HypnosCore` instance.
- **`async buildContext(options?)`** — Assembles a structured memory summary for the current project. The output is a plain-text or Markdown block suitable for injection into a system prompt. It must:
  1. Retrieve the most recent `N` interactions (default 20) from `interaction_logs`.
  2. Retrieve the top knowledge graph entities and their strongest relations from SurrealDB.
  3. Optionally run a semantic search against PGvector if a `currentContext` string is provided in `options` (e.g., the current file or user query).
  4. Format the results into a `[Project Memory]` block with subsections: `Recent Activity`, `Known Patterns`, `Relevant Context`.
  5. Respect a configurable `maxTokens` limit (default 2000) to avoid flooding the context window.
- **`async buildContextForFile(filePath)`** — Convenience method that calls `buildContext()` with the file path as the `currentContext`.

This class is consumed by the MCP server's Resource handler (Section 7.7).

### 7.6 `hypnos-core.js`

Export a `HypnosCore` class with the following contract:

- **`constructor(projectRoot)`** — Stores the project root. Creates instances of the PostgreSQL relational layer, `SurrealKnowledgeGraph`, and `VectorStore` (remote connects happen in `init()`).
- **`async init()`** — Initializes relational schemas (`ensureProjectSchema`, `ensureGlobalSchema`) and calls `connect()` on both `SurrealKnowledgeGraph` and `VectorStore`. Must be called after construction before using async methods.
- **`async recordInteraction(type, content, feedback?, metadata?)`** — Inserts a row into `interaction_logs`. `feedback` defaults to `0`, `metadata` defaults to `{}` and is stored as JSONB. **This is called by `EventListener`, not by the LLM.**
- **`async getInteractions(filter?)`** — Queries `interaction_logs` with optional filtering by `event_type`, `user_feedback`, or date range. Returns an array of row objects.
- **`async runSleepCycle()`** — The Generative Sleep consolidation process. This method is async because it interacts with SurrealDB and PGvector. It must:
  1. Query all `interaction_logs` where `user_feedback != 0`.
  2. For each validated log (`user_feedback = 1`): extract entity/relation/target triples and upsert them into SurrealDB via `SurrealKnowledgeGraph.upsertEntity()` and `upsertRelation()`.
  3. For each log: generate an embedding (placeholder/stub for now) and index the content into PGvector via `VectorStore.indexContent()`.
  4. Update `last_sleep_cycle` in the global database for this project.
  5. Return a summary object: `{ status: "success", processed: <count>, graphEntries: <count>, vectorized: <count> }`.
- **`async queryKnowledge(entity?)`** — Queries the SurrealDB knowledge graph via `SurrealKnowledgeGraph.queryByEntity()`. Returns all relations for the given entity.
- **`async traverseKnowledge(from, to, maxDepth?)`** — Delegates to `SurrealKnowledgeGraph.traversePath()`. Returns graph paths between two concepts.
- **`async semanticSearch(queryEmbedding, nResults?)`** — Searches PGvector via `VectorStore.search()`. Returns the top `nResults` (default 5) matches with their content and distances.
- **`async close()`** — Closes all connections: PostgreSQL pool, SurrealDB, and PGvector resources. Must be called during shutdown.

### 7.7 `mcp-server.js`

An MCP server (using `@modelcontextprotocol/sdk`) that implements the **hybrid architecture**: automatic Resources for the common path, explicit Tools only for deep queries.

**Startup:**

- Loads environment variables from the `.env` file in the project root using `dotenv`.
- Accepts the project root path from the client (via initialization params or environment variable `HYPNOS_PROJECT_ROOT`).
- Instantiates `HypnosCore` with that path and calls `await init()`.
- Instantiates `EventListener(hypnosCore)` and `ContextBuilder(hypnosCore)`.

**MCP Resources (automatic — no model action required):**

Resources are context that the MCP client fetches and injects into the model's prompt automatically on every turn. The model never decides to call these — the client does it transparently.

- **`hypnos://memory/context`** — Returns the output of `ContextBuilder.buildContext()`. The client reads this resource and appends it to the system prompt as a `[Project Memory]` block. This gives the model full awareness of recent activity, known patterns, and relevant knowledge without any tool call.
- **`hypnos://memory/context/{filePath}`** — Returns the output of `ContextBuilder.buildContextForFile(filePath)`. Used when the client knows which file the user is working on and wants file-specific memory.

**MCP Tools (on-demand — model decides to call):**

These are reserved for explicit, deep queries that go beyond the auto-injected context summary. Since there are only 3 tools (instead of 6), the model has a much higher chance of remembering them when needed.

- **`hypnos_search`** — calls `semanticSearch`. For when the model needs to find specific past interactions by semantic similarity.
- **`hypnos_traverse`** — calls `traverseKnowledge`. For when the model needs to explore relationship chains between concepts in the knowledge graph.
- **`hypnos_sleep`** — calls `runSleepCycle`. Triggers consolidation of recent interactions into long-term memory. Typically called manually or at the end of a session.

**Event ingestion (passive — driven by client, not model):**

The MCP server also exposes notification endpoints (or the client calls `EventListener` methods directly) for passive event capture:

- `notifications/file_changed` → `EventListener.onFileChange()`
- `notifications/suggestion_accepted` → `EventListener.onSuggestionAccepted()`
- `notifications/suggestion_rejected` → `EventListener.onSuggestionRejected()`
- `notifications/terminal_output` → `EventListener.onTerminalOutput()`
- `notifications/conversation_turn` → `EventListener.onConversationTurn()`

These are fire-and-forget. The client emits them; Hypnos records them. The model is never involved.

**Shutdown:**

- Handles graceful shutdown via `close()` on `HypnosCore`.

---

## 8. Testing Requirements

Use the **Node.js native test runner** (`node:test` + `node:assert`). Tests run via `node --test test/hypnos.test.js`.

### Required test cases:

### Core persistence tests:

1. **Project isolation** — Create two `HypnosCore` instances pointing at different temp directories. Record an interaction in one. Assert the other's `interaction_logs` table is empty.

2. **Interaction recording** — Record an interaction and verify it can be retrieved via `getInteractions`. Verify `metadata` round-trips correctly as JSON.

3. **Feedback filtering** — Record interactions with feedback values `1`, `-1`, and `0`. Query with a `user_feedback` filter and assert only the matching rows are returned.

### Passive event capture tests:

4. **EventListener file change** — Call `EventListener.onFileChange()` with a file path and diff. Assert a `code_fix` row is recorded in `interaction_logs` with the correct metadata.

5. **EventListener suggestion accepted** — Call `onSuggestionAccepted()`. Assert a `pattern` row is recorded with `user_feedback = 1`.

6. **EventListener suggestion rejected** — Call `onSuggestionRejected()`. Assert a `pattern` row is recorded with `user_feedback = -1`.

7. **EventListener conversation turn** — Call `onConversationTurn()` with a user message and an assistant message. Assert two rows are recorded.

### Sleep cycle & knowledge tests:

8. **Sleep cycle consolidation** — Record a validated interaction (`feedback=1`). Run `runSleepCycle()`. Assert the SurrealDB knowledge graph has at least one new entity and relation. Assert the returned summary has `status: "success"` and correct counts.

9. **Knowledge graph strength reinforcement** — Insert the same entity/relation/target pattern via two separate validated interactions. Run sleep cycle. Assert the corresponding `relates_to` edge in SurrealDB has `strength >= 2.0`.

10. **Graph traversal** — After a sleep cycle, call `traverseKnowledge(fromEntity, toEntity)` and assert a path is returned.

11. **Semantic search** — After a sleep cycle has indexed content, call `semanticSearch` with a query embedding and assert results are returned from PGvector.

### Context builder tests:

12. **Context build** — Record several interactions and run a sleep cycle. Call `ContextBuilder.buildContext()`. Assert the returned string contains a `[Project Memory]` block with `Recent Activity` and `Known Patterns` subsections.

13. **Context token limit** — Set `maxTokens` to a small value (e.g., 200). Assert the output is truncated to approximately that length.

14. **Context for file** — Call `buildContextForFile(filePath)`. Assert the output includes a `Relevant Context` subsection with content semantically related to the file.

### Lifecycle & error tests:

15. **Close lifecycle** — Call `close()` and assert that subsequent database operations throw.

16. **Invalid project root** — Pass an unwritable path and assert a meaningful error is thrown from the constructor.

17. **SurrealDB connection failure** — Provide invalid SurrealDB credentials and assert `init()` throws a meaningful connection error.

18. **PGvector connection failure** — Provide invalid PostgreSQL credentials and assert `init()` throws a meaningful connection error.

All tests must create temporary directories and clean them up in `t.after()`. Tests that interact with SurrealDB and PostgreSQL require those services to be running. Use environment variables to point tests at dedicated test databases/namespaces that are cleaned up after each run.

---

## 9. Configuration

A `.env` file in the **project root** holds all connection credentials and configuration. A `.env.example` file must be committed to the repo documenting every variable:

| Variable | Default | Description |
|---|---|---|
| `HYPNOS_PROJECT_ROOT` | (none, required) | Absolute path to the project root |
| | | |
| **SurrealDB** | | |
| `SURREAL_URL` | `http://127.0.0.1:8000` | SurrealDB server URL (HTTP or WebSocket) |
| `SURREAL_USER` | `root` | SurrealDB username |
| `SURREAL_PASS` | (none, required) | SurrealDB password |
| `SURREAL_NAMESPACE` | `hypnos` | Base namespace (project hash is appended automatically) |
| `SURREAL_DATABASE` | `knowledge` | Database name within the namespace |
| | | |
| **PostgreSQL + pgvector** | | |
| `PG_HOST` | `127.0.0.1` | PostgreSQL host |
| `PG_PORT` | `5432` | PostgreSQL port |
| `PG_USER` | `hypnos` | PostgreSQL username |
| `PG_PASS` | (none, required) | PostgreSQL password |
| `PG_DATABASE` | `hypnos` | PostgreSQL database name |
| | | |
| **General** | | |
| `HYPNOS_LOG_LEVEL` | `info` | Logging verbosity: `debug`, `info`, `warn`, `error` |
| `HYPNOS_CONTEXT_MAX_TOKENS` | `2000` | Maximum token budget for auto-injected `[Project Memory]` context |
| `HYPNOS_CONTEXT_RECENT_LIMIT` | `20` | Number of recent interactions to include in context summary |

Global relational schema settings:

| Variable | Default | Description |
|---|---|---|
| `HYPNOS_GLOBAL_SCHEMA` | `hypnos_global` | PostgreSQL schema name for global metadata (`projects`) |

All modules must read credentials via `process.env` after `dotenv` has loaded the `.env` file. Credentials must **never** be hardcoded.

---

## 10. `.gitignore` Requirements

The generated `.gitignore` must include at minimum:

```
node_modules/
.env
```

---

## 11. Future Development (Out of Scope for Initial Build)

These items are documented for architectural awareness but must NOT be implemented in the first version:

- **Fine-tuning pipeline.** Extract `user_feedback = -1` corrections, format as instruction/output pairs for LoRA fine-tuning via Unsloth or llama-cpp, and update a local model adapter (`.safetensors`).
- **Scheduled sleep cycles.** A cron-like scheduler that triggers `runSleepCycle()` automatically (e.g., daily at idle time).
- **Cross-project knowledge federation.** Optional opt-in sharing of anonymized patterns across projects via the global store.
- **Embedding model selection.** Allow configuring which embedding model is used for PGvector (local vs. API-based). The initial build uses a stub/placeholder for embedding generation.

---

## 12. Agent Implementation Checklist

The implementing agent must complete these steps in order:

1. Initialize the npm project (`package.json`).
2. Install all dependencies from Section 3 (`@modelcontextprotocol/sdk`, `surrealdb`, `pg`, `dotenv`).
3. Create `.gitignore` per Section 10.
4. Create `.env.example` per Section 9.
5. Implement `src/database.js` per Section 7.1.
6. Implement `src/surreal-client.js` per Section 7.2.
7. Implement `src/vector-store.js` per Section 7.3.
8. Implement `src/event-listener.js` per Section 7.4.
9. Implement `src/context-builder.js` per Section 7.5.
10. Implement `src/hypnos-core.js` per Section 7.6.
11. Implement `src/mcp-server.js` per Section 7.7 (Resources + Tools + Notifications).
12. Implement `test/hypnos.test.js` per Section 8.
13. Ensure SurrealDB and PostgreSQL (with pgvector) are running.
14. Run the test suite and ensure all 18 tests pass.
15. Verify project schemas and global metadata schema are created and cleaned up correctly.