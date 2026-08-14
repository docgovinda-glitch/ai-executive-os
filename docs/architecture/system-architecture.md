# System Architecture

## High-Level Architecture

The AI Executive OS follows a **layered architecture** with a strict separation between the user-facing UI, the application services (Rust backend), and the operating system. The design is **local-first** — all processing and data storage happen on the user's device.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    DESKTOP APPLICATION                                   │
│                    (apps/desktop)                                       │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  UI Layer — React 19 + TypeScript + Vite                           │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐  │  │
│  │  │  Chat View  │  │Agent View │  │Plugin View │  │Settings View│  │  │
│  │  └────────────┘  └────────────┘  └────────────┘  └────────────┘  │  │
│  │                                                                     │  │
│  │  State: Zustand (global) + React Query (server/cache)              │  │
│  │  Events: Tauri Event Listener (real-time updates)                  │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│        │                               │                              │
│        │ Tauri Invoke (permission-  │                              │
│        │ checked gateway)             │                              │
│        ▼                               ▼                              │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  Service Layer — Rust (Tauri Backend)                               │  │
│  │                                                                     │  │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │  │
│  │  │Agent Engine │ │AI Provider  │ │Tool Manager │ │Plugin Manager │  │  │
│  │  │             │ │Manager      │ │             │ │             │  │  │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘   │  │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │  │
│  │  │Task Manager │ │Security     │ │Storage      │ │Config       │   │  │
│  │  │             │ │Manager      │ │Manager      │ │Manager      │   │  │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘   │  │
│  │                                                                     │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │  │
│  │  │  MCP Client & Built-in Tool Runtime                            │ │  │
│  │  └─────────────────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│        │                               │                              │
│        │                               │                              │
│        │ Tauri Events (streaming,     │                              │
│        │ tool lifecycle)              │                              │
│        ▼                               │                              │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  OS Layer — System APIs (macOS, Windows, Linux)                          │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐         │
│  │File System  │ │Network Stack│ │Process Mgmt │ │Input/Output │         │
│  │(std::fs)    │ │(reqwest)    │ │(std::proc)  │ │(OS APIs)    │         │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘         │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  External Data Sources                                              │  │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │  │
│  │  │LLM APIs     │ │Cloud Services│ │MCP Servers  │ │Local Models │   │  │
│  │  │(OpenAI etc) │ │(Google, MS) │ │(plugins)    │ │(Ollama)    │   │  │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘   │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

## Layer Descriptions

### 1. UI Layer (React 19)

The frontend is a single-page application (SPA) built with React 19, TypeScript, and Vite 7. It runs inside a Tauri webview and communicates with the Rust backend exclusively through:

- **Tauri Invoke API:** Request-response calls to registered commands.
- **Tauri Events:** Real-time streaming updates (agent thought process, tool execution, LLM streaming tokens).

#### Key UI Components

| Component | Responsibility |
|-----------|----------------|
| `ChatView` | Main conversation interface — message list, user input, LLM streaming |
| `AgentView` | Real-time visualization of agent reasoning, tool calls, and execution results |
| `TaskView` | Task list, status, scheduling, and history |
| `PluginView` | Plugin management — install, configure, enable/disable, view registered tools |
| `SettingsView` | Model selection, provider configuration, preferences |
| `ConversationsView` | Conversation history, search, and management |
| `SystemView` | System diagnostics, logs, resource usage |

### 2. Service Layer (Rust)

The backend is written in Rust and compiles to a native binary using Tauri 2. It provides all application services through a single **Command Gateway** that enforces permissions and dispatches to internal service methods.

#### Core Services

| Service | Module | Responsibility |
|---------|--------|----------------|
| **Agent Engine** | `agent/src/` | LLM orchestration: planning, reasoning, tool selection, execution loop (ReAct) |
| **AI Provider Manager** | `llm/src/` | Abstract LLM provider trait; adapters for OpenAI, Anthropic, Google, Ollama, etc. |
| **Tool Manager** | `tools/src/` | Tool registry; built-in tools + MCP-provided tools; tool execution sandboxing |
| **Plugin Manager** | `plugins/src/` | MCP client; plugin process lifecycle; capability registration |
| **Task Manager** | `tasks/src/` | Background task execution; task state persistence; scheduling |
| **Security Manager** | `security/src/` | Credential storage (OS keychain); permission validation; audit logging |
| **Storage Manager** | `storage/src/` | SQLite database access; migrations; backup/restore |
| **Config Manager** | `config/src/` | Application settings; provider configurations; feature flags |

#### Command Gateway

All frontend-to-backend communication flows through a single authenticated and permission-checked gateway:

```
Frontend → invoke("gateway", { command: "...", params: {...} })
         → Gateway validates command + permissions
         → Gateway dispatches to service method
         → Result serialized and returned to frontend
```

### 3. OS Layer

The application interacts with the operating system through both Tauri's built-in APIs and direct Rust standard library calls:

- **File System:** `std::fs` (direct) + Tauri filesystem plugin (permission-gated)
- **Network:** `reqwest` crate (HTTP/HTTPS)
- **Processes:** `std::process` + `tokio::process` for async process management
- **Input Simulation:** Platform-specific APIs (macOS CGEvent, Windows SendInput, Linux X11/Wayland)
- **Screen Capture:** Platform-specific APIs (macOS CGDisplay, Windows Graphics Capture API, Linux screenshot tools)
- **Notifications:** Tauri notification plugin
- **Keychain:** `keyring` crate with OS-native backends

### 4. External Integrations

The system integrates with external services through:

- **LLM APIs:** HTTPS REST/SSE to provider endpoints (OpenAI, Anthropic, etc.)
- **Cloud Services:** OAuth 2.0 flows + API calls (Google Workspace, Microsoft 365, etc.)
- **MCP Plugins:** JSON-RPC 2.0 over stdio transport
- **Local Models:** HTTP API (Ollama, LM Studio) or direct inference

## Data Flow

### Request-Response Flow (Frontend → Backend)

1. User interacts with the UI (clicks a button, sends a message).
2. Frontend calls `invoke("gateway", { command: "sendMessage", params: { ... } })`.
3. Tauri routes the invoke to the Rust `#[tauri::command] fn gateway(...)`.
4. Gateway validates the command against the user's granted capabilities.
5. Gateway dispatches to the appropriate service (e.g., `ConversationService::send_message`).
6. Service orchestrates LLM call, tool execution, etc.
7. Result is serialized and returned through Tauri's IPC channel.
8. Frontend receives the result and renders the updated state.

### Streaming Flow (Agent → Frontend)

1. Agent Engine starts processing a user message.
2. As the LLM generates tokens, they are sent as Tauri events (`agent:token`).
3. Frontend listens for these events and appends to the message stream.
4. Tool execution events (`tool:start`, `tool:result`) are emitted in real-time.
5. Agent reasoning steps (`agent:thought`, `agent:action`) are streamed as events.

### Plugin Tool Flow (MCP)

1. Plugin Manager starts an MCP server process at startup.
2. MCP client negotiates the protocol (initialize handshake).
3. MCP server announces its tools, resources, and prompts.
4. Plugin Manager registers these with the Tool Manager.
5. When the agent needs a tool, Tool Manager routes to either:
   - Built-in tool (direct Rust call)
   - MCP tool (forward to MCP server via JSON-RPC)

## Architectural Patterns

| Pattern | Usage |
|---------|-------|
| **Layered Architecture** | Clear separation of UI, Service, and OS layers |
| **Gateway/Adapter** | Command gateway centralizes all backend communication |
| **Strategy Pattern** | LLM provider adapters implement a common trait |
| **Observer Pattern** | Tauri events let the frontend observe agent activity |
| **Registry Pattern** | Tool and plugin registries allow dynamic discovery |
| **Repository Pattern** | Storage Manager abstracts database access behind traits |
| **Command Pattern** | All operations are invokable commands with typed parameters |
| **Factory Pattern** | Provider/Plugin/Service factories handle instantiation |

## Cross-Cutting Concerns

| Concern | Implementation |
|---------|----------------|
| **Logging** | Structured logging via `tracing` crate; logs stored locally |
| **Configuration** | TOML config files + SQLite settings table |
| **Error Handling** | Custom error types with `thiserror`; user-friendly messages |
| **Async Runtime** | Tokio (multi-threaded) for all async operations |
| **Internationalization** | i18n keys in Rust; translation files in frontend |
| **Metrics** | In-memory metrics for performance monitoring (not telemetry) |
