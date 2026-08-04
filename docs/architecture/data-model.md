# Data Model

## Core Data Stores

The system relies on a local SQLite database to persist conversation state, task state, provider configuration, and security audit data.

### Primary Tables

| Table | Purpose |
|------|---------|
| `conversations` | Stores thread metadata, title, and timestamps |
| `messages` | Stores prompt/response turns, roles, and metadata |
| `tasks` | Tracks agent work items, status, and plan steps |
| `providers` | Stores model configuration and active defaults |
| `credentials` | Stores secret references or encrypted vault records |
| `audit_log` | Stores policy-relevant execution records |

## Storage Strategy

- Structured metadata stays in SQLite.
- Large binary artifacts stay on disk.
- Sensitive secrets remain in the OS keychain or equivalent secure storage.
