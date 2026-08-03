# ADR-004: Abstract Multi-LLM Provider Integration

## Status

Proposed

## Context

The AI Executive OS must support multiple AI providers to:
- Allow users to choose their preferred LLM (OpenAI, Anthropic, Google, local models)
- Enable fallback when a provider is unavailable or rate-limited
- Support local LLMs for privacy-sensitive operations
- Allow per-task model selection (e.g., cheaper model for simple tasks, reasoning model for complex planning)

The abstraction must handle:
- Different API formats (Chat Completions API, Claude Messages API, Ollama, etc.)
- Different authentication methods (API keys, OAuth, local)
- Different token/streaming capabilities
- Different cost and latency characteristics

### Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| **Direct provider SDK integration** | Full feature access; native SDKs | Vendor lock-in; each provider needs custom code; can't swap models dynamically |
| **LiteLLM (proxy)** | Unified API; 100+ models; well-maintained | Requires running a Python service; adds a server component; conflicts with "serverless" |
| **OpenRouter** | Unified API; single key; many models | Centralized; not local-first; requires internet |
| **Custom unified LLM trait** | Full control; no external proxy; local-first possible | Must implement adapters per provider |
| **LangChain / LlamaIndex** | Rich tooling; multi-provider | Python-heavy; adds complexity; not Rust-native |

## Decision

Implement a **custom unified LLM trait** in Rust with per-provider adapters.

### LLM Provider Interface (Rust Trait)

```rust
/// Unified interface for all LLM providers
#[async_trait]
pub trait LLMProvider: Send + Sync + 'static {
    /// Provider name (e.g., "openai", "anthropic", "ollama", "google")
    fn name(&self) -> &str;

    /// Provider display name
    fn display_name(&self) -> String;

    /// Available models for this provider
    fn available_models(&self) -> &[LLMModel];

    /// Check if the provider is properly configured
    async fn is_available(&self) -> bool;

    /// Send a chat completion request with streaming
    async fn chat_stream(
        &self,
        request: ChatRequest,
    ) -> Result<Pin<Box<dyn tokio_stream::Stream<Item = Result<ChatChunk>>>, LLMError>;

    /// Send a chat completion request (non-streaming)
    async fn chat(
        &self,
        request: ChatRequest,
    ) -> Result<ChatResponse, LLMError>;

    /// Count tokens in a message
    fn count_tokens(&self, messages: &[ChatMessage]) -> usize;

    /// Health check / latency probe
    async fn health_check(&self) -> Result<ProviderHealth, LLMError>;
}
```

### Supported Adapters (MVP → Future)

| Priority | Provider | Adapter | Authentication |
|----------|----------|---------|----------------|
| P1 | OpenAI | `llm-openai` | API Key (keychain) |
| P1 | Anthropic | `llm-anthropic` | API Key (keychain) |
| P2 | Google | `llm-google` | API Key (keychain) |
| P2 | Ollama | `llm-ollama` | Local (no auth) |
| P2 | LM Studio | `llm-lmstudio` | Local HTTP (no auth) |
| P3 | Grok | `llm-grok` | API Key (keychain) |
| P3 | Azure OpenAI | `llm-azure` | API Key (keychain) |
| P3 | Local GGUF | `llm-ggml` | Direct model file |

### Provider Manager

- Manages all configured providers and their health status.
- Routes requests to the appropriate provider based on:
  - User's selected default model
  - Fallback chain (if primary fails, try next)
  - Cost/latency preferences
  - Model capability requirements (e.g., vision, large context)

### Local LLM Strategy

- Local models are treated as first-class providers.
- The adapter communicates via HTTP (Ollama API, LM Studio API) or via direct GGUF inference (using `llama-cpp` Rust crate).
- Local model binaries are downloaded to `Local Data/plugins/models/` (user opt-in).
- Vision and tool-calling support varies by local model; the agent engine handles capability mismatches.

## Consequences

### Positive
- Full control over the abstraction layer.
- No external proxy or server required — truly serverless.
- Each provider adapter is isolated and independently testable.
- Easy to add new providers by implementing the trait.
- Fallback chains provide resilience.
- Local-first: providers can be entirely local (Ollama, LM Studio, GGUF).

### Negative
- Must implement and maintain per-provider adapters.
- Token counting per model requires per-provider tuning.
- Cannot easily benefit from upstream bug fixes in a shared library (mitigated by keeping adapters thin).

### Neutral
- The trait-based design allows for future consolidation if a suitable unified Rust crate emerges.
- Provider configuration (API keys, base URLs) is managed by the Security/Secret Manager.
