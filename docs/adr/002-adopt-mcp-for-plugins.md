# ADR-002: Adopt Model Context Protocol (MCP) for Plugin Extensibility

## Status

Proposed

## Context

The AI Executive OS must be extensible — users need to add new tools, data sources, and capabilities without modifying the core application. The extensibility model must:

- Support tools (functions the agent can call)
- Support resources (data sources the agent can read)
- Support prompts (templated prompts)
- Allow local plugin development
- Support community plugins
- Be language-agnostic (plugins can be written in any supported language)
- Have a well-defined, stable specification

### Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| **Custom JSON-RPC protocol** | Full control, no external dependencies | Reinventing the wheel; no ecosystem; maintenance burden |
| **OpenAI Function Calling only** | Simple, widely understood | No dynamic plugin discovery; no resources/prompts; vendor lock-in |
| **A2A (Google Agent2Agent)** | Standardized agent communication | Overkill for plugin system; focused on inter-agent communication |
| **Direct Rust plugin loading (cdylib)** | High performance | Platform-specific compilation; no cross-language support |
| **MCP (Model Context Protocol)** | Emerging standard; growing ecosystem; supports tools, resources, prompts; language-agnostic; backed by Anthropic | New standard, ecosystem still maturing; Rust implementation (RMCP) is relatively new |

## Decision

Adopt **MCP (Model Context Protocol)** as the primary plugin extensibility model. MCP servers can be:
- **Local executables** (compiled Rust/Python/Node.js binaries on the user's machine)
- **Community plugins** (installed from a plugin registry or manually)
- **Development plugins** (local dev servers running locally)

### MCP Integration Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  AI Executive OS (Host)                       │
│                                                             │
│  ┌──────────┐   ┌────────────────────────────────────────┐ │
│  │ MCP      │   │ MCP Client (Rust: rmcp or tarpc-mcp)  │ │
│  │ Plugin   │   │ Registers tools, resources, prompts    │ │
│  │ Manager  │◄──┤ Routes tool calls to appropriate        │ │
│  │          │   │ MCP server process                       │ │
│  └──────────┘   └──────────────┬─────────────────────────┘ │
│                               │                            │
│  ┌────────────────────────────┼──────────────────────────┐ │
│  │  Agent Engine               │                           │ │
│  │  (LLM orchestration)        │                          │ │
│  └────────────────────────────┼──────────────────────────┘ │
│                               │                            │
│  ┌────────────────────────────┼──────────────────────────┐ │
│  │  Tool Manager               │                           │ │
│  │  (Built-in + MCP tools)     │                          │ │
│  └────────────────────────────┼──────────────────────────┘ │
│                               │                            │
│  ┌────────────────────────────┼──────────────────────────┐ │
│  │ MCP Server Process (e.g.   │  ┌─────────────────────┐  │ │
│  │ Python/Go/Rust plugin)      ├──│ JSON-RPC over stdio  │  │ │
│  └────────────────────────────┼──┘                       │ │
│                               │                          │ │
│  ┌────────────────────────────┼──────────────────────────┐ │
│  │ MCP Server Process (e.g.   │  ┌─────────────────────┐  │ │
│  │ Node.js plugin)             ├──│ JSON-RPC over stdio  │  │ │
│  └────────────────────────────┼──┘                       │ │
└───────────────────────────────┴────────────────────────────┘
```

### Plugin Discovery

- Plugins are discovered from:
  1. Built-in plugins (shipped with the app)
  2. User-installed local plugins (configured in `~/.ai-exec-os/plugins/`)
  3. System-wide plugins (optional, OS-specific paths)
- Each plugin declares a manifest (mcp.json) with: name, description, command, arguments, env, capabilities

### Lifecycle Management

- Plugin Manager spawns plugin processes as child processes
- Manages process lifecycle: start, health check, restart on crash, graceful shutdown
- Tools and resources are registered with the Tool Manager at startup
- Plugin processes are sandboxed via OS process isolation

## Consequences

### Positive
- Leverages a growing open standard with community momentum
- Supports tools, resources, and prompts in a unified interface
- Language-agnostic: plugins can be written in Python, Go, Rust, Node.js, etc.
- Dynamic discovery: new capabilities added at runtime without restart
- Future-proof: if MCP evolves, the agent's tool interface remains stable

### Negative
- MCP is a relatively new standard; the Rust client ecosystem (rmcp) is maturing
- Plugin process management adds complexity (crash recovery, health monitoring)
- Security: plugins run as subprocesses and can access the file system — must be sandboxed/trusted

### Neutral
- The core built-in tools will also be exposed via the same tool interface as MCP tools, ensuring uniformity
- Users must review plugin permissions before installation (similar to VS Code extensions)
