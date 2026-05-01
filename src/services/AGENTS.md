# AGENTS.md — src/services/

## OVERVIEW

Core services layer: memory CRUD, auto-capture, embedding, tagging, context formatting, web API, and web server.

## STRUCTURE

```
services/
├── ai/                    # AI provider factory (OpenAI, Anthropic, opencode)
├── api-handlers.ts        # All web API route handlers
├── auto-capture.ts        # Idle-triggered memory extraction from conversation
├── cleanup-service.ts     # Orphan/unlinked memory cleanup
├── client.ts              # LocalMemoryClient — central memory CRUD API
├── context.ts             # formatContextForPrompt() — memory→prompt injection
├── deduplication-service.ts # Exact/near-duplicate detection
├── embedding.ts           # EmbeddingService singleton (local @huggingface/transformers or remote API)
├── migration-service.ts   # Embedding model dimension migration
├── privacy.ts             # stripPrivateContent, isFullyPrivate
├── secret-resolver.ts     # file:// and env:// URI resolution
├── sqlite/                # connection-manager, shard-manager, vector-search
├── tags.ts                # getTags() — project isolation via sha256(tag)
├── user-memory-learning.ts # Periodic user profile analysis
├── user-profile/          # UserProfileManager, profile-context
├── user-prompt/           # UserPromptManager — prompt persistence + capture tracking
├── vector-backends/       # USearch + ExactScan implementations
├── web-server.ts          # Bun.serve HTTP server with port takeover
└── web-server-worker.ts   # Web server worker (inlined into web-server.ts)
```

## WHERE TO LOOK

| Task                           | Location                                                            |
| ------------------------------ | ------------------------------------------------------------------- |
| Add new memory operation       | `client.ts` → `LocalMemoryClient`                                   |
| Change auto-capture behavior   | `auto-capture.ts` → `performAutoCapture()`                          |
| Modify prompt injection format | `context.ts` → `formatContextForPrompt()`                           |
| Tag format / project isolation | `tags.ts` → `getTags()`, `getProjectIdentity()`                     |
| Embedding model / cache        | `embedding.ts` → `EmbeddingService`                                 |
| New web API route              | `web-server.ts` → `handleRequest()` + `api-handlers.ts`             |
| Shard routing logic            | `client.ts` → `resolveScopeValue()`, `shardManager.getWriteShard()` |
| Privacy filtering              | `privacy.ts`                                                        |

## CONVENTIONS

- All API handlers return `{ success, data?, error? }` shape
- Memory IDs: `mem_${Date.now()}_${Math.random().toString(36).substring(2, 11)}`
- Container tags: `${CONFIG.containerTagPrefix}_project_${sha256(projectIdentity)}`
- Two embedding modes: local (`@huggingface/transformers`) or remote (`embeddingApiUrl` + `embeddingApiKey`)
- `embeddingService` is a global singleton via `Symbol.for("opencode-mem.embedding.instance")`
- Opencode provider preferred for auto-capture (no separate API key needed)
- Web server uses Bun.serve; non-owners attempt port takeover with jitter (500-1500ms)

## ANTI-PATTERNS

- **Tag parsing in delete/search**: `client.ts:225-237` iterates ALL shards (user + project) to find one memory — O(n) scan, not shard-local
- **No transaction around delete+insert**: `api-handlers.ts:409-440` delete then re-insert in separate calls, no atomic tx
- **Global mutable migration state**: `api-handlers.ts:963-970` module-level `migrationProgress` variable shared across requests
- **Unbounded prompt search**: `userPromptManager.searchPrompts()` called with no limit in `handleSearch` — can return unlimited results
- **embedding.ts cache eviction**: Removes oldest entry when cache full but `Float32Array` may still be referenced by callers
