# ADR-001: Use Tauri 2 + React + TypeScript as the Desktop Stack

## Status

Accepted

## Context

The project requires a desktop application that can:
- Run natively on macOS, Windows, and Linux
- Interact with the OS at a low level (input simulation, screen capture, process management)
- Provide a rich, responsive web-based UI
- Maintain a small bundle size and minimal resource footprint
- Support secure IPC between UI and system-level operations

Existing alternatives considered:

- **Electron**: Large bundle size (~200MB+), high memory usage. Not suitable for a "lean, local-first" assistant.
- **Tauri 1**: Single-window focus, fewer plugin capabilities.
- **Native macOS/Windows/Linux apps**: Would require separate codebases per platform.
- **Web-only PWA**: Cannot access OS-level APIs (input simulation, screenshot, process management).
- **Rust + egui**: Good for native UI but less ergonomic for complex, data-rich chat interfaces.

## Decision

Use **Tauri 2** (Rust backend) with **React 19** + **TypeScript** (Vite 7 build system) as the desktop application framework.

### Rationale

1. **Cross-platform:** Tauri compiles to native binaries on macOS, Windows, and Linux from a single codebase.
2. **Performance:** Rust backend with WASM-free native execution. UI runs in a lightweight webview (not Chromium).
3. **Security:** Tauri enforces a strict permissions model — the frontend cannot access OS APIs directly. Only explicitly registered `#[tauri::command]` handlers can invoke Rust code.
4. **Bundle size:** Typical Tauri apps are ~10-30MB, compared to 100MB+ for Electron.
5. **Ecosystem:** The existing scaffold already uses this stack (see `apps/desktop/src-tauri/Cargo.toml`, `apps/desktop/package.json`).
6. **Rust ecosystem:** The Rust crate ecosystem provides strong libraries for:
   - Async runtime (Tokio)
   - HTTP clients (Reqwest)
   - Database (SQLx, Diesel, or Rusqlite)
   - Crypto (RustCrypto, keyring)
   - MCP client integration (RMCP)
   - Computer vision (Tesseract, image processing)

## Consequences

### Positive
- Unified codebase across all desktop platforms.
- Small bundle size and low memory footprint.
- Strong security boundary between UI and OS.
- Rust provides memory safety without GC overhead.
- Existing project scaffold already set up with this stack.

### Negative
- Rust has a steeper learning curve than JavaScript/TypeScript.
- Tauri mobile support is still maturing (not a priority for v1).
- Inter-process communication via async Rust requires careful design.

### Neutral
- Frontend is still JavaScript/TypeScript, so web developers can contribute to the UI layer.
- Backend logic in Rust will require cross-functional team knowledge.

## Alternatives Considered

### Electron + Node.js
- Rejected: Bundle size (~200MB+) and memory usage are incompatible with a "lean, local-first" philosophy.

### Tauri 1
- Rejected: Tauri 2 provides better plugin architecture, improved security model, and first-class mobile support.

### Native Rust UI (Iced / egui)
- Rejected: Complex for rich chat interfaces. Web technologies (React) are more productive for UI development.

### Flutter Desktop
- Rejected: Would introduce a completely new framework and toolchain, increasing complexity without clear benefits over Tauri.
