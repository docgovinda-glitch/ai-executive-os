# ADR-006: Granular Tauri Permissions Model and Command Gateway

## Status

Proposed

## Context

Tauri 2 enforces a permissions-based security model. By default, the frontend (webview) cannot access any OS-level APIs — it communicates with Rust only through explicitly registered `#[tauri::command]` handlers and the Tauri Event system.

For the AI Executive OS, we need to balance:
- **Security:** The frontend must never have direct OS access. All OS interactions must be mediated by Rust.
- **Flexibility:** The agent engine needs rich OS capabilities (file system, processes, input simulation, screenshots).
- **Granular permission control:** Users should be able to grant/revoke permissions per capability (e.g., "allow screenshot" but not "allow keystroke injection").

### Tauri Permissions Overview

Tauri's permission system works by:
1. Each `@tauri-apps/cli` command or plugin declares permissions.
2. A `capabilities/*.json` file lists which permissions are granted to which windows.
3. The frontend requests capabilities, but only those explicitly allowed by a capability file can be used.

### Current State (from `/apps/desktop/src-tauri/capabilities/default.json`)

```json
{
  "identifier": "default",
  "description": "Capability for the main window",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "opener:default"
  ]
}
```

This is the default Tauri template — only basic window opening is permitted. We need a much richer permission model.

## Decision

1. **Replace the default capability** with a granular permission set organized by feature.
2. **Implement a Command Gateway pattern** in Rust: all frontend-to-backend communication goes through a single `gateway` command that validates permissions and dispatches to internal service methods.
3. **Define capability categories** matching the agent's tool domains.

### Capability Categories

| Capability ID | Permissions | Used By | Description |
|--------------|-------------|---------|-------------|
| `fs:default` | file-system read/write permissions | File System Tool, Plugin Manager | Read/write user files, plugin manifests |
| `fs:screenshots` | Screenshot capture | Computer Tool | Capture screen for vision models |
| `os:input` | Input simulation (mouse, keyboard) | Computer Tool | Control mouse/keyboard |
| `os:process` | Process management | System Tool | List, spawn, kill processes |
| `os:network` | Network state | System Tool | Network status, interfaces |
| `browser:automation` | Browser control | Browser Tool | Drive browser via Playwright/Chromium |
| `plugins:manage` | Plugin lifecycle | Plugin Manager | Install, start, stop plugins |
| `ai:providers` | LLM provider config | AI Provider Manager | Manage API keys, provider settings |
| `tasks:schedule` | Task scheduling | Task Manager | Background task execution |
| `settings:manage` | App settings | Settings Service | Read/write user preferences |

### Command Gateway Pattern

```
Frontend (React)
    │
    │  invoke("gateway", { command: "readFile", params: {...} })
    ▼
┌─────────────────────────────────────────────────────┐
│  Gateway Command (Rust)                             │
│  1. Validate requested command against capabilities │
│  2. Deserialize params into typed struct             │
│  3. Dispatch to service method                       │
│  4. Serialize result back to frontend                │
└─────────────────────────────────────────────────────┘
```

This pattern ensures:
- Every command is permission-checked.
- All dispatch logic is centralized.
- Adding new commands requires updating one place.
- Audit logging can be added centrally.

### Type Safety Between Frontend and Backend

- Rust command signatures are used to generate TypeScript types via `tauri-spectator`.
- The frontend imports these generated types, ensuring type-safe contracts.
- All command parameters are validated on the Rust side using `serde` + `validator`.

## Consequences

### Positive
- Fine-grained permission control at the OS level.
- Centralized command dispatch simplifies auditing and logging.
- Type-safe contracts between Rust and TypeScript.
- Security boundary between frontend and OS is enforced by Tarsi itself.
- Users can review and selectively grant permissions.

### Negative
- More complex capability files to maintain.
- Every new command requires a permission declaration.
- The gateway pattern adds a small dispatch overhead (negligible).

### Neutral
- The permission model can be extended per-platform (macOS permissions dialog, Windows UAC, Linux AppArmor/SELinux).
- Future: Dynamic permission requests (frontend asks for permission, Rust shows OS dialog).
