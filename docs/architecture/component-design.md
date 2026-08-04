# Component Design

## Purpose

This document describes the major internal components, their responsibilities, and the responsibilities they expose to the rest of the system.

## Planned Modules

### UI Modules

- Chat surface and streaming renderer
- Agent timeline and tool trace view
- Plugin management surface
- Settings and provider configuration

### Runtime Modules

- Agent orchestrator
- Provider manager
- Tool registry and sandbox executor
- Plugin lifecycle controller
- Storage and migration layer
- Security and permission service

## Design Principles

1. Keep the command surface typed and permission-gated.
2. Prefer small, testable services over a monolithic runtime.
3. Allow plugins to extend the system without changing the core engine.

## MVP Component Boundary

The initial build should focus on a single desktop shell with the minimum set of runtime services needed to support chat, model routing, tool execution, and secure local storage.
