# Review: Upstream PR #2887 — More Idiomatic Effect Runtime Primitives

**PR**: https://github.com/pingdotgg/t3code/pull/2887
**Author**: app/cursor (bot)
**Status**: Merged
**Files changed**: 4 (+45 / ~lines)

## Summary

This PR makes three independent changes to follow Effect-first patterns more closely:

1. **`OpenCodeRuntimeError`**: Converts from `Data.TaggedError` to `Schema.TaggedErrorClass`, gaining serialization support and a schema-based guard.
2. **Timeout input**: Changes `timeoutMs?: number` → `timeout?: Duration.Input` on `OpenCodeRuntimeShape`, using `Duration` for formatting and `Effect.timeoutOption`.
3. **Provider status cache**: Moves from raw `JSON.stringify` to `fromJsonStringPretty` (Schema-based JSON encoding). Excludes transient `updateState` from cache snapshots.

## Fork Compatibility Assessment

| Change | Compatible? | Breaking constructor sites | Notes |
|--------|-------------|---------------------------|-------|
| `Data.TaggedError` → `Schema.TaggedErrorClass` | **BREAKING** | ~12 `new OpenCodeRuntimeError({…})` across 5 files | Constructor changes to `.make({…})`; `.is()` guard stays compatible |
| `timeoutMs: number` → `Duration.Input` | **BREAKING** | Interface + `DEFAULT_OPENCODE_SERVER_TIMEOUT_MS` + error message | `Duration` import missing; `Effect.timeoutOption` accepts both forms |
| `fromJsonStringPretty` in cache | **COMPATIBLE** | 0 | Already exported from `@t3tools/shared/schemaJson` |
| `updateState` exclusion from cache | **COMPATIBLE** | 0 | Destructure pattern identical to existing fork code |

### Details

#### 1. TaggedError migration

Current fork uses `Data.TaggedError` (line 52 of `opencodeRuntime.ts`):

```ts
export class OpenCodeRuntimeError extends Data.TaggedError("OpenCodeRuntimeError")<{
  readonly operation: string;
  readonly cause?: unknown;
  readonly detail: string;
}> {
  static readonly is = (u: unknown): u is OpenCodeRuntimeError =>
    P.isTagged(u, OPENCODE_RUNTIME_ERROR_TAG);
}
```

Upstream replaces with `Schema.TaggedErrorClass`:

```ts
export class OpenCodeRuntimeError extends Schema.TaggedErrorClass<OpenCodeRuntimeError>()(
  "OpenCodeRuntimeError",
  {
    operation: Schema.String,
    detail: Schema.String,
    cause: Schema.optional(Schema.Defect),
  },
) {
  override get message() { return this.detail; }
}
export const isOpenCodeRuntimeError = Schema.is(OpenCodeRuntimeError);
```

**Impact**: All `new OpenCodeRuntimeError({ operation, detail, cause })` calls break — `Schema.TaggedErrorClass` uses `.make({…})` or schema decoding. The exported `isOpenCodeRuntimeError` guard replaces `OpenCodeRuntimeError.is`. The `P.isTagged` pattern is dropped; `Schema.is` provides equivalent behavior.

Affected call sites (must migrate constructor syntax):
- `opencodeRuntime.ts`: ~8 internal error construction sites
- `OpenCodeAdapter.ts`: 2 sites
- `OpenCodeAdapter.test.ts`: 1 site
- `OpenCodeProvider.test.ts`: 2 sites
- `OpenCodeTextGeneration.test.ts`: 1 site

#### 2. Duration timeout

Current fork uses `timeoutMs?: number` with `DEFAULT_OPENCODE_SERVER_TIMEOUT_MS = 5_000`. Upstream changes to `timeout?: Duration.Input` with `DEFAULT_OPENCODE_SERVER_TIMEOUT = Duration.seconds(5)`.

`Effect.timeoutOption()` already accepts both `number` (interpreted as ms) and `Duration`. This means the behavioral semantics are preserved — a 5-second timeout is a 5-second timeout regardless of representation.

The `Duration` import (`import * as Duration from "effect/Duration"`) must be added.

#### 3. Cache serialization

`fromJsonStringPretty` already exists in our fork at `packages/shared/src/schemaJson.ts:155`. The upstream change replaces:

```ts
// Before: raw JSON.stringify
contents: `${JSON.stringify(cacheableProvider, null, 2)}\n`,
```

with:

```ts
// After: Schema.encodeEffect via fromJsonStringPretty
const contents = yield* encodeProviderStatusCache(cacheableProvider);
yield* writeFileStringAtomically({ filePath, contents: `${contents}\n` });
```

This changes `writeProviderStatusCache` from synchronous to an Effect generator. This is the only change that touches the call signature pattern. The function already returns an `Effect`, so wrapping in `Effect.gen` is additive, not breaking.

## Stability Impact on OpenCode Provider

**No stability risk.** All three changes are structural refinements, not behavioral:

- **Timeout**: Identical 5-second timeout, same `Effect.timeoutOption` path. `Duration.Input` is a superset of `number` — existing callers passing numbers would still work.
- **Error handling**: `Schema.TaggedErrorClass` provides the same `.is()` discrimination and carries the same fields. Error propagation through `toProcessError` → `ProviderAdapterProcessError` is unchanged.
- **Cache encoding**: `fromJsonStringPretty` delegates to `Schema.fromJsonString` with a formatting wrapper. The encoded JSON shape is equivalent to `JSON.stringify(obj, null, 2)` for all current `ServerProvider` schemas. The `updateState` exclusion is a correctness fix (transient state shouldn't be cached), not a behavior change.

## Recommendation

**Apply.** This PR is a low-risk idiomatic cleanup. The changes are mechanical and well-scoped. The migration involves ~12 constructor call sites + Duration import — all localized to the opencode runtime subsystem. No changes to the provider stability contract.

Priority order if cherry-picking:
1. `fromJsonStringPretty` + cache updateState exclusion (lowest risk, highest correctness)
2. `Duration.Input` timeout (middle risk, mechanical)
3. `Schema.TaggedErrorClass` migration (highest risk due to constructor changes, but purely mechanical)
