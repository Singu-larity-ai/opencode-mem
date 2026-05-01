# AI Provider Abstraction Layer

## OVERVIEW

Factory-pattern provider system for multi-AI backend support (OpenAI, Anthropic, Google Gemini) with session state management and structured tool output.

## STRUCTURE

```
src/services/ai/
├── ai-provider-factory.ts        # Factory: createProvider(), getSupportedProviders()
├── provider-config.ts            # buildMemoryProviderConfig() for external APIs
├── opencode-provider.ts          # OpenCode OAuth/API key bridge + structured output
├── providers/
│   ├── base-provider.ts          # BaseAIProvider abstract class + ToolCallResult
│   ├── openai-chat-completion.ts # OpenAI /chat/completions provider
│   ├── openai-responses.ts      # OpenAI Responses API provider
│   ├── anthropic-messages.ts     # Anthropic /messages API provider
│   └── google-gemini.ts          # Google Gemini provider
├── session/
│   ├── ai-session-manager.ts     # AISessionManager: SQLite-backed session/message store
│   └── session-types.ts         # AISession, AIMessage, AIProviderType
├── tools/
│   └── tool-schema.ts            # ChatCompletionTool schema + ToolSchemaConverter
└── validators/
    └── user-profile-validator.ts # Profile data validation
```

## WHERE TO LOOK

| Task                    | Location                                | Notes                                          |
| ----------------------- | --------------------------------------- | ---------------------------------------------- |
| Add new provider        | `providers/` + `ai-provider-factory.ts` | Extend BaseAIProvider, register in factory     |
| Session state           | `session/ai-session-manager.ts`         | SQLite tables: ai_sessions, ai_messages        |
| Tool schema format      | `tools/tool-schema.ts`                  | ToolSchemaConverter for Anthropic format       |
| OpenCode OAuth bridge   | `opencode-provider.ts`                  | createOAuthFetch(), createOpencodeAIProvider() |
| Structured output       | `opencode-provider.ts:255`              | generateStructuredOutput() using AI SDK        |
| Config for external API | `provider-config.ts`                    | buildMemoryProviderConfig()                    |

## CONVENTIONS

- All providers extend `BaseAIProvider` → implement `executeToolCall()`, `getProviderName()`, `supportsSession()`
- Providers receive `AISessionManager` instance for session tracking
- Tool call loop: maxIterations + iterationTimeout with AbortController timeout
- Messages stored with sequence numbers; tool incomplete sequences filtered
- OAuth token refresh: auto-refresh on expiry, persisted back to auth.json
- MCP tool names prefixed with `mcp_` when forwarding through OAuth proxy

## ANTI-PATTERNS

- Do NOT call API without session context — always use session manager for message history
- Do NOT skip `filterIncompleteToolCallSequences()` — results in corrupted tool call sequences
- Do NOT hardcode provider credentials — use opencode-provider.ts OAuth bridge or provider-config.ts
- Do NOT forget AbortController timeout cleanup — causes resource leaks in iteration loops
