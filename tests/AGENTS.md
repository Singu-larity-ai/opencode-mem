# TEST SUITE — opencode-mem

## OVERVIEW

Bun test runner (`bun:test`) with flat file structure. 17 test files + `vector-backends/` subdirectory.

## STRUCTURE

```
tests/
├── *.test.ts              # Flat — no subdirectories except vector-backends/
├── vector-backends/
│   └── *.test.ts
```

## WHERE TO LOOK

| Need                    | Look                                                |
| ----------------------- | --------------------------------------------------- |
| Config defaults/merging | `config.test.ts`                                    |
| Privacy stripping       | `privacy.test.ts`                                   |
| Vector backend behavior | `vector-backends/exact-scan-backend.test.ts`        |
| Provider fetch mocking  | `ai-provider-config.test.ts`                        |
| Session-dependent code  | `ai-provider-config.test.ts` → `FakeSessionManager` |

## CONVENTIONS

- **Naming**: `*.test.ts`, one file per module/unit under test
- **Imports**: `import { describe, it, expect, beforeEach, afterEach, afterAll } from "bun:test"`
- **Assertion style**: `expect(value).toBe(...)`, `expect(arr).toEqual([...])`, `expect(x).toContain(...)`
- **Describe nesting**: top-level `describe("ModuleName")`, sub-levels `describe("methodName")`
- **Isolation — env**: `afterAll` restores `process.env.HOME` / `USERPROFILE` after tests mutate it
- **Isolation — fetch**: `afterEach` restores `globalThis.fetch = originalFetch`
- **Isolation — temp dirs**: `afterEach` pops `tempDirs[]` array, calls `rmSync(dir, { recursive: true, force: true })`
- **Temp dir pattern**: `mkdtempSync(join(tmpdir(), "prefix-"))`
- **Fake classes**: inline `class FakeXxx` for complex deps (session manager, etc.) — use `as any` cast
- **Fetch mocking**: replace `globalThis.fetch` with async arrow returning mock `Response` object
- **Lifecycle order**: `beforeEach` setup → `afterEach` teardown → `afterAll` final restore

## ANTI-PATTERNS

- **No `beforeAll` for env mutation**: tests that set `process.env.HOME` use `afterAll` restore; prefer per-test `beforeEach` setup when possible
- **No snapshot tests**: use explicit `toEqual` with concrete expected values
- \*\*No `describe.skip`/` it.skip`: temporarily disabled tests should be deleted, not skipped
