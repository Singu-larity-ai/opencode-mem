# SQLite Storage Layer

## OVERVIEW

Sharded SQLite storage with WAL mode, per-project/user database files, and USearch/ExactScan vector search backends.

## STRUCTURE

```
src/services/sqlite/
├── connection-manager.ts   # Connection pool, WAL mode, checkpointing
├── shard-manager.ts        # Per-project/user shard routing + lifecycle
├── sqlite-bootstrap.ts     # bun:sqlite lazy init wrapper
├── vector-search.ts        # Vector similarity search + CRUD
└── types.ts                # ShardInfo, MemoryRecord, SearchResult
```

## WHERE TO LOOK

| Task                     | Location                      | Notes                                                                    |
| ------------------------ | ----------------------------- | ------------------------------------------------------------------------ |
| Add sharding logic       | `shard-manager.ts`            | `getWriteShard()`, `createShard()`                                       |
| Connection lifecycle     | `connection-manager.ts`       | `getConnection()`, `closeAll()`, `checkpointAll()`                       |
| WAL/checkpoint config    | `connection-manager.ts:14-18` | PRAGMA settings applied at init                                          |
| Vector insert/search     | `vector-search.ts`            | `insertVector()`, `searchInShard()`, USearch primary                     |
| Vector backend factory   | `vector-search.ts:22-24`      | `createVectorBackend()` → USearch or ExactScan                           |
| Fallback on search error | `vector-search.ts:103-126`    | Degrades to ExactScan on backend failure                                 |
| Table schema             | `shard-manager.ts:156-181`    | `memories` table + 4 indexes                                             |
| Embedding storage        | `shard-manager.ts:159`        | `vector BLOB NOT NULL` — Float32Array serialized to Uint8Array           |
| Shard routing            | `shard-manager.ts:223-252`    | Routes to active shard, rotates when `vectorCount >= maxVectorsPerShard` |

## CONVENTIONS

- **bun:sqlite only** — uses `bun:sqlite` runtime, lazy-loaded via `sqlite-bootstrap.ts`
- **WAL mode** — `PRAGMA journal_mode = WAL`, `synchronous = NORMAL`
- **Checkpoint on idle** — `checkpointAll()` called by connection manager
- **Shards are append-only** — new shard created when `vectorCount >= maxVectorsPerShard`; old shard marked `is_active = 0`
- **Dual vector storage** — both `vector` (content) and `tags_vector` (tags) stored as BLOB in SQLite; USearch/ExactScan indexed separately
- **Metadata shard** — `metadata.db` at `storagePath/` root tracks all shard registry (not in `users/` or `projects/` dirs)
- **Path stored relative** — `db_path` in shards table uses relative paths; resolved via `resolveStoredPath()`
- **Prepared statements only** — no raw string interpolation in queries

## ANTI-PATTERNS

- **SQL injection in LIKE** — `vector-search.ts:299` uses string interpolation in LIKE clause: `metadata LIKE '%"sessionID":"${sessionID}"%'` — do NOT replicate this pattern
- **String concatenation for paths** — `shard-manager.ts:277` uses `LIKE '%' || ?` for path lookup — fragile on Windows; prefer exact match or `likelihood()` function
