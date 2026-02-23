# Hypnos

Hypnos is a localized memory and “Generative Sleep” sidecar service for AI coding agents (VS Code Copilot, Antigravity, OpenCode, or any MCP-compatible client). It provides long-term, project-scoped memory so agents improve over time within a specific codebase.

## Why Hypnos

Large language models often forget to call low-priority memory tools during task execution. Hypnos removes the model from the critical path for common memory operations:

- Write path: passive event listeners capture interactions automatically.
- Read path: MCP Resources auto-inject project memory context.
- Deep query path: MCP Tools are available on-demand.
- Maintenance path: sleep cycle can be triggered manually or administratively.

## Core Principles

- Project isolation: no memory leakage across repositories.
- Passive-first: memory capture and recall are automatic.
- Offline-first: local persistence only for core memory.
- Lifecycle safety: every connection must be closable via `close()`.
- Concurrency safety: SQLite uses WAL mode.
- Sync/async honesty: synchronous SQLite methods are not marked async.

## Architecture

Hypnos uses three persistence layers:

1. Project store (SQLite)
- Path: `<project-root>/.hypnos/memory.db`
- Purpose: interaction logs

2. Global metadata store (SQLite)
- Path: `%APPDATA%/hypnos/global.db` (Windows) or `~/.config/hypnos/global.db` (Linux/macOS)
- Purpose: known projects, `last_accessed`, `last_sleep_cycle`

3. Knowledge + semantic stores
- SurrealDB: project-scoped knowledge graph (namespace/database per project hash)
- PostgreSQL + pgvector: project-scoped semantic embeddings and ANN search

## Planned Repository Structure

```text
hypnos/
├── src/
│   ├── database.js
│   ├── surreal-client.js
│   ├── vector-store.js
│   ├── event-listener.js
│   ├── context-builder.js
│   ├── hypnos-core.js
│   └── mcp-server.js
├── test/
│   └── hypnos.test.js
├── .gitignore
├── .env.example
├── package.json
├── DEFINITION.md
└── README.md
```

## Prerequisites

- Node.js 18+
- SurrealDB running
- PostgreSQL running with `pgvector` extension available

## Setup

1. Install dependencies

```bash
npm install
```

2. Create your environment file

```bash
cp .env.example .env
```

On Windows PowerShell:

```powershell
Copy-Item .env.example .env
```

3. Fill required environment variables (especially credentials and project root).

## Environment Variables

Required/important variables:

- `HYPNOS_PROJECT_ROOT` (required)
- `SURREAL_URL` (default: `http://127.0.0.1:8000`)
- `SURREAL_USER` (default: `root`)
- `SURREAL_PASS` (required)
- `SURREAL_NAMESPACE` (default: `hypnos`)
- `SURREAL_DATABASE` (default: `knowledge`)
- `PG_HOST` (default: `127.0.0.1`)
- `PG_PORT` (default: `5432`)
- `PG_USER` (default: `hypnos`)
- `PG_PASS` (required)
- `PG_DATABASE` (default: `hypnos`)
- `HYPNOS_LOG_LEVEL` (default: `info`)
- `HYPNOS_CONTEXT_MAX_TOKENS` (default: `2000`)
- `HYPNOS_CONTEXT_RECENT_LIMIT` (default: `20`)

See `.env.example` for the complete template.

## Running

Start the MCP server:

```bash
npm run start
```

## Testing

Run the Node.js native test suite:

```bash
npm run test
```

Expected test command:

```bash
node --test test/hypnos.test.js
```

The complete suite includes core persistence, passive event capture, sleep-cycle consolidation, graph traversal, semantic search, context building, lifecycle behavior, and connection-failure tests.

## MCP Surface

Resources (automatic context injection):

- `hypnos://memory/context`
- `hypnos://memory/context/{filePath}`

Tools (on-demand deep queries):

- `hypnos_search`
- `hypnos_traverse`
- `hypnos_sleep`

Passive notifications (client-driven):

- `notifications/file_changed`
- `notifications/suggestion_accepted`
- `notifications/suggestion_rejected`
- `notifications/terminal_output`
- `notifications/conversation_turn`

## Operational Notes

- Add `.hypnos/` to `.gitignore`.
- Never hardcode credentials; read from `process.env`.
- Ensure all SQLite connections enable WAL mode.
- Always call `HypnosCore.close()` on shutdown.

## Current Scope (v1)

Included:

- Local project memory logging
- SurrealDB knowledge graph consolidation
- PGvector semantic indexing/search
- MCP resources/tools/notifications for the hybrid architecture

Explicitly out of scope for v1:

- Fine-tuning pipeline
- Scheduled sleep cycles
- Cross-project federation
- Embedding model/provider selection framework

## Canonical Specification

The full technical specification is in [DEFINITION.md](DEFINITION.md).
