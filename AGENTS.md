# PROJECT KNOWLEDGE BASE

**Generated:** 2026-04-30
**Commit:** 40508eb
**Branch:** main

## OVERVIEW

OpenCode plugin providing persistent memory for AI coding agents via local SQLite + USearch vector database. TypeScript/Bun ESM plugin with multi-provider AI support and a web UI for memory management.

## STRUCTURE

```
opencode-mem/
├── src/
│   ├── index.ts            # Plugin entry — hooks + tool registration
│   ├── plugin.ts           # Thin re-export wrapper (PluginModule)
│   ├── config.ts           # Config loading, defaults, validation (582 lines)
│   ├── services/           # Business logic core
│   │   ├── ai/             # AI provider abstraction (factory pattern)
│   │   ├── sqlite/         # Storage layer (connection, sharding, vector search)
│   │   ├── vector-backends/ # USearch + ExactScan backends
│   │   ├── user-profile/   # User profile learning & management
│   │   └── user-prompt/    # Prompt persistence
│   ├── types/              # Shared type definitions
│   └── web/                # Static web UI (vanilla JS/CSS/HTML)
├── tests/                  # Bun test suite (bun:test)
└── .github/workflows/      # Tag-triggered release → npm publish
```

## WHERE TO LOOK

| Task                     | Location                                     | Notes                                            |
| ------------------------ | -------------------------------------------- | ------------------------------------------------ |
| Add new memory operation | `src/services/client.ts`                     | Central memory client API                        |
| Add AI provider          | `src/services/ai/providers/`                 | Implement BaseProvider, register in factory      |
| Change config defaults   | `src/config.ts` → `DEFAULTS`                 | All defaults centralized there                   |
| Vector search logic      | `src/services/sqlite/vector-search.ts`       | SQLite-backed vector queries                     |
| Web UI changes           | `src/web/app.js`, `styles.css`               | Vanilla JS, CDN deps (lucide, marked, dompurify) |
| Plugin hooks             | `src/index.ts` → return object               | `chat.message`, `event`, `tool`                  |
| Test a service           | `tests/`                                     | Flat structure, `*.test.ts`                      |
| Config resolution        | `src/config.ts` → `initConfig()`             | Global + project config merge                    |
| Embedding models         | `src/config.ts` → `getEmbeddingDimensions()` | 30+ models with dimension map                    |
| Privacy filtering        | `src/services/privacy.ts`                    | `stripPrivateContent`, `isFullyPrivate`          |

## CODE MAP

| Symbol                       | Type   | Location                               | Role                               |
| ---------------------------- | ------ | -------------------------------------- | ---------------------------------- |
| `OpenCodeMemPlugin`          | Export | `src/index.ts:20`                      | Main plugin factory                |
| `CONFIG`                     | Var    | `src/config.ts:567`                    | Resolved config (global singleton) |
| `initConfig`                 | Fn     | `src/config.ts:569`                    | Loads & merges config from files   |
| `buildConfig`                | Fn     | `src/config.ts:483`                    | Builds config with defaults        |
| `memoryClient`               | Export | `src/services/client.ts`               | Central memory CRUD API            |
| `performAutoCapture`         | Export | `src/services/auto-capture.ts`         | Idle-triggered memory extraction   |
| `performUserProfileLearning` | Export | `src/services/user-memory-learning.ts` | User profile analysis              |
| `startWebServer`             | Export | `src/services/web-server.ts`           | HTTP server for web UI             |

## CONVENTIONS

- **Runtime**: Bun (primary). Build uses `bunx tsc`.
- **Module system**: ESM only (`"type": "module"`, `verbatimModuleSyntax: true`)
- **Imports**: Always use `.js` extensions in import paths (ESM requirement)
- **Formatting**: Prettier — double quotes, semicolons, 2-space indent, 100 char width
- **Pre-commit**: `bun run typecheck && bunx lint-staged` (via Husky)
- **Config format**: JSONC (`opencode-mem.jsonc`) at `~/.config/opencode/`
- **Data storage**: `~/.opencode-mem/data/` (SQLite databases)
- **Tool responses**: All JSON strings (`JSON.stringify({...})`)
- **Error handling**: Errors caught and returned as `{ success: false, error: String(error) }`
- **Logging**: Custom `log()` from `src/services/logger.ts` (not console.log)
- **Secret resolution**: `file://` and `env://` prefixes supported for API keys

## ANTI-PATTERNS (THIS PROJECT)

- **SQL injection risk**: `src/services/sqlite/vector-search.ts:299` uses string interpolation in LIKE clause — do NOT replicate this pattern
- **Heavy `any` casting**: 30+ `as any` casts across codebase — prefer proper interfaces
- **innerHTML in web UI**: `src/web/app.js` uses innerHTML — sanitize all user-controlled data
- **Dynamic imports for circular deps**: `src/index.ts` uses `await import(...)` to break cycles — prefer restructuring
- **No ESLint**: Only Prettier + TypeScript strict mode for quality enforcement

## UNIQUE STYLES

- **Tag-based project isolation**: Memories tagged by project hash (`tags.ts` → `getTags()`)
- **Dual vector backend**: USearch-first with automatic ExactScan fallback (`vectorBackend` config)
- **Session-aware injection**: Memories injected on first user message or after compaction (`injectOn` config)
- **Auto-capture**: Idle-triggered AI extraction of conversation insights (`session.idle` event)
- **Compaction restoration**: On `session.compacted` event, memories re-injected as synthetic parts
- **Web server takeover**: Multiple plugin instances coordinate single web server ownership
- **Global warmup**: Plugin warmup runs once via `Symbol.for("opencode-mem.plugin.warmedup")`

## COMMANDS

```bash
bun install                    # Install dependencies
bun run build                  # Compile TS + copy web assets to dist/
bun run dev                    # Watch mode (tsc --watch)
bun run typecheck              # Type check without emit
bun run format                 # Format with Prettier
bun run format:check           # Check formatting
bun test                       # Run tests (Bun built-in)
```

## NOTES

- **No test script in package.json**: Run `bun test` directly
- **CI runs only on tag push** (`v*`): No PR validation pipeline
- **Embedding models**: Local via `@huggingface/transformers`, no API call needed for embeddings
- **API key formats**: Plain string, `file://path`, or `env://ENV_VAR` for all API keys
- **Web UI deps**: Loaded from CDN (lucide, marked, dompurify, jsonrepair) — not bundled
- **SQLite WAL mode**: Checkpoint via `connection-manager.checkpointAll()` on idle
