# ADR-005: React 19 for the Frontend UI

## Status

Accepted

## Context

The desktop application requires a rich, interactive user interface for:
- Chat-based conversation with the AI assistant
- Real-time display of agent actions and tool execution
- Plugin management and configuration UI
- Settings and preferences
- Task history and logs
- Context visualization (files, browser state, resources)

The UI must be:
- Responsive and performant (the app should feel "snappy")
- Maintainable with a component-based architecture
- Accessible (keyboard navigation, screen reader support)
- Themable (light/dark modes)
- Buildable into a Tauri webview bundle

### Alternatives Considered

- **React 18**: Mature, well-understood, but lacks the latest ergonomics (actions, useFormStatus, etc.)
- **SvelteKit**: Smaller bundles, but less familiar to the team; fewer UI libraries
- **Vue 3 + Vite**: Good DX, but different component paradigm
- **Vanilla JS + HTMX**: Minimal, but insufficient for a complex chat/agent UI
- **Iced (Rust GUI)**: Native, but too low-level for rich UIs

## Decision

Use **React 19** with TypeScript, built via Vite 7, rendered in a Tauri webview.

### Key Technologies

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Framework | React 19 | Concurrent rendering, Actions, useOptimistic, useFormState |
| Language | TypeScript 5.8 | Type safety across the full stack |
| Build | Vite 7 | Fast HMR, optimized production builds |
| Styling | Tailwind CSS | Utility-first, themable, zero-runtime |
| State | Zustand + React Query | Global state (Zustand) + server/cache state (React Query) |
| Routing | React Router v7 | Type-safe client routing for multi-view (chat, settings, plugins) |
| Forms | React Hook Form + Zod | Type-safe form validation |
| Testing | Vitest + React Testing Library | Unit and component tests |
| Linting | ESLint + Prettier | Code quality and consistency |

### Component Architecture

The frontend is organized by **feature domains**:

```
src/
├── app/              # App shell (router, providers, layout)
├── features/
│   ├── chat/         # Conversation UI (messages, input, streaming)
│   ├── agent/        # Agent action visualization (tool calls, logs)
│   ├── tasks/        # Task list, scheduling, execution status
│   ├── plugins/      # Plugin management (install, configure, list)
│   ├── settings/     # Model selection, preferences, providers
│   ├── credentials/  # API key management UI
│   ├── conversations/ # Conversation list, search, history
│   └── system/       # System status, logs, diagnostics
├── shared/           # Shared components, hooks, utilities, types
│   ├── components/   # Reusable UI primitives
│   ├── hooks/        # Custom hooks
│   ├── lib/          # Utilities (formatting, etc.)
│   └── types/        # Shared types (generated from Rust API)
├── styles/           # Global CSS, theme tokens
└── routes/           # Route-level lazy-loaded components
```

### Tauri Bridge Integration

- Frontend communicates with Rust backend exclusively via **Tauri Invoke API** (`@tauri-apps/api/core`).
- All backend commands are typed using TypeScript types generated from Rust command signatures (see [ADR-006](./006-tauri-permissions-model.md)).
- Frontend requests async data via React Query, which calls Tauri commands.
- Real-time updates (agent streaming, tool execution) use Tauri **events** (`@tauri-apps/api/event`).

## Consequences

### Positive
- React 19 provides the latest ergonomics and performance improvements.
- Vite provides extremely fast development iteration.
- Tailwind CSS enables rapid UI development with consistent theming.
- Feature-based organization scales well as the app grows.
- Type-safe contracts between Rust and TypeScript via generated types.

### Negative
- React's learning curve for junior developers (though React 19 is more ergonomic).
- Frequent React major versions can introduce breaking changes.
- Tauri webview has limited access to browser APIs (mitigated by Tauri plugins).

### Neutral
- The frontend is decoupled from the Rust backend through well-defined invoke contracts.
- Migration to a different frontend framework in the future would only require reimplementing the invoke layer.
