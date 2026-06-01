# Review: Upstream PR #2891 — Load OpenCode Skills

**PR**: https://github.com/pingdotgg/t3code/pull/2891
**Author**: razzeee (Kolja)
**Status**: Open
**Files changed**: 2 (+29 / -3 lines)

## Summary

This PR adds skill discovery to the OpenCode provider. It fetches skills from the
OpenCode server via `client.app.skills()`, maps them into the `ServerProviderSkill`
contract format, and includes them in the provider status snapshot.

The two-file change touches the OpenCode runtime layer (fetch + inventory) and the
provider layer (mapping + integration into `buildServerProvider`).

## Detailed Change Analysis

### 1. `opencodeRuntime.ts` — Skill Fetching

Adds three pieces:

```ts
// New interface
export interface OpenCodeSkill {
  readonly name: string;
  readonly description: string;
  readonly location: string;
}

// New field on OpenCodeInventory
export interface OpenCodeInventory {
  readonly providerList: ProviderListResponse;
  readonly agents: ReadonlyArray<Agent>;
+ readonly skills: ReadonlyArray<OpenCodeSkill>;
}

// New loader (parallel with providers + agents)
const loadSkills = (client: OpencodeClient) =>
  runOpenCodeSdk("app.skills", () => client.app.skills()).pipe(
    Effect.map((result) =>
      (result.data ?? []).map(({ name, description, location }) =>
        ({ name, description, location })
      )
    ),
  );
```

**Design notes:**

- Uses the existing `runOpenCodeSdk` wrapper — consistent error handling with all
  other SDK calls. Errors surface as `OpenCodeRuntimeError` with span
  `opencode.app.skills` in telemetry.
- `result.data ?? []` is defensive against null responses (matching the
  `loadAgents` pattern a few lines above).
- Only destructures `name`, `description`, `location` from the SDK response.
  The SDK also returns `content` (the skill body), but `ServerProviderSkill`
  doesn't need it — this is a metadata-only mapping. Correct scope.
- Third argument to `Effect.all` in `loadOpenCodeInventory` — runs
  `loadProviders`, `loadAgents`, `loadSkills` with `concurrency: "unbounded"`.
  No serialization penalty; skills fetch in parallel.

### 2. `OpenCodeProvider.ts` — Provider Integration

```ts
function mapOpenCodeSkills(
  skills: ReadonlyArray<OpenCodeSkill>,
): ReadonlyArray<ServerProviderSkill> {
  return skills.map((skill) => ({
    name: skill.name,
    description: skill.description || undefined,
    path: skill.location,
    enabled: true,
  }));
}
```

Then in `checkOpenCodeProviderStatus`, after the inventory is loaded:

```ts
const skills = mapOpenCodeSkills(inventoryExit.value.skills);
return buildServerProvider({
  // ... existing fields ...
+ skills,
});
```

**Mapping correctness:**
| OpenCodeSkill | → | ServerProviderSkill | Notes |
|---|---|---|---|
| `name: string` | → | `name: TrimmedNonEmptyString` | Direct passthrough |
| `description: string` | → | `description?: string` | `|| undefined` converts empty string to undefined |
| `location: string` | → | `path: string` | Renamed — one is filesystem path, other is logical path |
| _(none)_ | → | `enabled: true` | Hardcoded — a skill fetched from the server is always enabled |

The `|| undefined` guard is important because the SDK may return `description: ""`.
`Schema.optional(Schema.String)` on `ServerProviderSkill.description` would fail
on empty string. The fallback to `undefined` avoids a decode error.

## Fork Compatibility Assessment

| Change                                   | Compatible? | Breaking? | Notes                                                                                                                    |
| ---------------------------------------- | ----------- | --------- | ------------------------------------------------------------------------------------------------------------------------ |
| `OpenCodeSkill` interface                | **YES**     | No        | New interface, no conflicts                                                                                              |
| `skills` on `OpenCodeInventory`          | **YES**     | No        | Additive field; consumers that destructure `{ providerList, agents }` are unaffected                                     |
| `loadSkills` function                    | **YES**     | No        | New; uses existing `runOpenCodeSdk` pattern                                                                              |
| `loadOpenCodeInventory` update           | **YES**     | No        | Adds third `Effect.all` argument; destructuring changed from `[providerList, agents]` → `[providerList, agents, skills]` |
| `ServerProviderSkill` import             | **YES**     | No        | Already defined in `@t3tools/contracts`                                                                                  |
| `OpenCodeSkill` import                   | **YES**     | No        | Type-only import from sibling module                                                                                     |
| `mapOpenCodeSkills` function             | **YES**     | No        | New; pure mapping function                                                                                               |
| `skills` in `buildServerProvider`        | **YES**     | No        | `ServerProvider.skills` has default `[]` via `Schema.withDecodingDefault(Effect.succeed([]))`                            |
| SDK `client.app.skills()`                | **YES**     | No        | Present in `@opencode-ai/sdk/v2` at `gen/sdk.gen.d.ts:133`                                                               |
| `makePendingOpenCodeProvider` call sites | **YES**     | No        | Not affected — uses `buildServerProvider` without `skills`, default `[]` applies                                         |

### No Call Site Breakage

The `buildServerProvider` function accepts a partial `ServerProvider` object.
Because `skills` defaults to `[]` via `Schema.withDecodingDefault`, existing
call sites that don't pass `skills` are unaffected. Only `checkOpenCodeProviderStatus`
adds skills — `makePendingOpenCodeProvider` (two call sites) and the
version-too-old error path continue to default to `[]`.

### No Type Cascade

`OpenCodeInventory.skills` addition doesn't break any other code because:

1. `flattenOpenCodeModels(input: OpenCodeInventory)` destructures only
   `input.providerList` and `input.agents` — it ignores `skills`.
2. The return type of `loadOpenCodeInventory` changes from 2-tuple to 3-tuple
   in the `Effect.all` destructuring, but this is internal to the runtime
   implementation — the public interface return type is `OpenCodeInventory`,
   which now carries the additional field.

## Does This Fix "Opencode Skills not found"?

**Yes.** The `ServerProvider.skills` field was already wired into the provider
contract (introduced in a prior PR), but nothing was populating it for the
OpenCode provider. Without this PR:

1. `ServerProvider.skills` always decoded as `[]` (the default).
2. Any UI or consumer expecting skills from `ServerProvider.skills` would
   find nothing.
3. Skills discovered by OpenCode's server were invisible to T3 Code.

With this PR:

1. `checkOpenCodeProviderStatus` loads skills from the running OpenCode server.
2. Skills are mapped to the contract format and included in the provider snapshot.
3. Consumers (including the web UI) see the skills as part of the provider status.

The fix is targeted: it only touches the OpenCode provider layer. No changes to
the contracts, client-runtime, or apps/web packages.

## Risks and Edge Cases

### Risk: Empty or missing skills

**Mitigated.** `result.data ?? []` handles null/undefined responses. An empty
skills array is semantically correct — the provider has no skills to expose.

### Risk: SDK error during skill fetch

**Mitigated.** `runOpenCodeSdk` wraps the call in `Effect.tryPromise` with
error mapping to `OpenCodeRuntimeError`. If `client.app.skills()` throws, the
error propagates through `checkOpenCodeProviderStatus` → `inventoryExit._tag === "Failure"`
→ `fallback()`. The provider status shows a probe error rather than crashing.

### Risk: Skill with empty description string

**Mitigated.** `skill.description || undefined` converts empty string to
`undefined`, preventing `ServerProviderSkill` decode failure on the optional
`description` field.

### Risk: Skill with empty name

**Not explicitly mitigated.** `ServerProviderSkill.name` is
`TrimmedNonEmptyString` — a skill with an empty name would cause a decode
failure. However, this would be an OpenCode server bug (it shouldn't return
nameless skills). The PR doesn't add a filter for this, but the impact is
contained — the `buildServerProvider` call would fail, and the probe error
fallback would surface it.

**Recommendation:** Consider adding `skills.filter(s => s.name.trim().length > 0)`
as a defensive measure, but this is optional and shouldn't block the PR.

## Recommendation

**Apply.** This is a clean, minimal, low-risk addition. All changes are
additive — no existing code paths are altered. The pattern follows existing
conventions (`loadAgents` for the loader shape, `runOpenCodeSdk` for error
wrapping, parallel `Effect.all` for concurrency). The `mapOpenCodeSkills`
mapping is correct and handles the edge cases that matter.

**Priority:** Apply directly. No cherry-picking needed — all changes are in
two files with no cross-cutting concerns.

**Merge confidence:** High. The diff is small (32 lines changed), well-scoped
to the OpenCode subsystem, and doesn't touch any other provider, the contracts,
or the UI layer.
