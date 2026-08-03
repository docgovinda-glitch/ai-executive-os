# ADR-003: Local-First Architecture with SQLite as the Primary Store

## Status

Accepted

## Context

The AI Executive OS is designed as a **local-first**, **serverless** application. This means:

- All user data, conversations, credentials, and agent state must be stored locally on the user's device.
- No cloud backend is required to operate.
- Data must be available offline.
- Strong privacy: no telemetry, no analytics, no data exfiltration by default.

We need a local storage solution that:
- Is embedded (no separate server process)
- Supports concurrent reads (and serialized writes) from Rust async tasks
- Supports structured querying for conversations, tasks, settings, and plugin configs
- Is cross-platform (macOS, Windows, Linux)
- Has minimal external dependencies

### Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| **SQLite** | Embedded, zero-config, ACID, cross-platform, mature, supports SQL queries, very stable | Write locking; not ideal for very high write throughput (not a concern here) |
| **IndexedDB (via webview)** | Already available in webview context | Not accessible from Rust directly; web-only; limited querying |
| **Sled (Rust embedded KV)** | Native Rust, async-friendly, embedded | Less mature; no SQL; harder for structured queries |
| **RocksDB** | High performance, embedded | Overkill for this scale; more complex setup |
| **JSON files** | Simple, human-readable | No concurrency control; no querying; corruption risk |
| **PostgreSQL (local)** | Full SQL, robust | Requires a server process; too heavy for "embedded" local-first |

## Decision

Use **SQLite** as the primary local data store, accessed from Rust via the `sqlx` or `rusqlite` crate.

### Data Storage Strategy

```
Local Data Directory (platform-specific)
├── data/                     # SQLite database
│   └── ai-exec-os.sqlite
├── plugins/                  # Installed MCP plugins
├── credentials/              # Encrypted credential blobs (keychain-backed)
├── cache/                    # Transient cache (screenshots, OCR results)
├── logs/                     # Agent execution logs
└── exports/                  # User-exported data
```

### Database Schema Overview

| Table | Purpose |
|-------|---------|
| `conversations` | Conversation sessions |
| `messages` | Individual messages (user, assistant, tool results) |
| `tasks` | Agent-planned sub-tasks and their execution state |
| `tool_executions` | History of tool calls and results |
| `settings` | Key-value settings (UI preferences, AI config) |
| `plugins` | Installed plugin metadata |
| `plugin_capabilities` | Registered tools/resources per plugin |
| `credentials_meta` | Metadata for stored credentials (no secrets stored here) |
| `audit_log` | Security-relevant events |

### Concurrency Model

- SQLite with WAL (Write-Ahead Logging) mode for concurrent reads.
- A single writer pattern via a Tokio-mutexed connection pool.
- All database operations go through a centralized **Storage Manager** (Rust service) that ensures:
  - Migrations are applied automatically on startup.
  - Schema versioning is tracked.
  - All access is async-safe.

### Credentials

- Secrets (API keys, OAuth tokens) are **not** stored in SQLite.
- They are stored in the **OS keychain** (via the `keyring` crate):
  - macOS: Keychain Services
  - Windows: Credential Manager (DPAPI-backed)
  - Linux: Secret Service API (libsecret) or `pass` fallback
- SQLite stores only a key identifier and metadata (last used, associated plugin/provider).

## Consequences

### Positive
- Zero-install: SQLite is an embedded library, no server needed.
- ACID transactions ensure data integrity.
- SQL querying enables rich analytics on conversations and tasks.
- WAL mode provides good read concurrency.
- OS keychain integration provides hardware-backed secret storage.
- Cross-platform with identical behavior.
- Easy to back up (single file copy).

### Negative
- SQLite file can grow over time; periodic vacuum/reindex may be needed.
- Not easily horizontally scalable (not needed for a desktop app).
- Schema migrations must be carefully managed across versions.

### Neutral
- The Storage Manager will abstract SQLite behind a clean Rust API trait, allowing future migration if needed.
- Backup/restore features can be built on top of the single-file database.
