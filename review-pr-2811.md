# Review: Upstream PR #2811 — Load OpenCode skills from `.agents/skills/` directory

**PR**: [pingdotgg/t3code#2811](https://github.com/pingdotgg/t3code/pull/2811)
**Branch**: `feat/opencode-skills`
**Status**: OPEN (not merged)
**Files**: `OpenCodeProvider.ts` (+125/-2), `OpenCodeDriver.ts` (+11/-2)
**Reviewer**: gastown.nux (polecat, t3-29f.2)

---

## Summary

PR #2811 adds local filesystem-based skill loading for the OpenCode provider by reading `.agents/skills/<dir>/SKILL.md` from the workspace directory. This mirrors the Codex driver's behavior (which loads skills via `skills/list` RPC) for the OpenCode driver, which previously had no skill discovery at all.

## Fork Compatibility

**Verdict: Applies cleanly with minor adjustments.**

Our fork (`duncan4123/t3code`) matches the PR's base branch structure:

| File                  | PR base commit | Our fork                             | Match? |
| --------------------- | -------------- | ------------------------------------ | ------ |
| `OpenCodeProvider.ts` | `dea95c990`    | Identical structure, same signatures | Yes    |
| `OpenCodeDriver.ts`   | `816e8b70f`    | Identical structure                  | Yes    |

Both files in our fork share the same imports, function signatures, and call patterns. The PR's changes are purely additive — no existing code is modified, only new functions are added and existing calls are extended with additional parameters.

**Required adjustments for our fork:**

1. `OpenCodeDriver.ts` already imports `Path` and `FileSystem` (lines 19-20). The PR adds `Layer` and `NodeFileSystem` imports — these are new dependencies that must be installed/available. `NodeFileSystem` is from `@effect/platform-node`, which is already a dependency of `apps/server`.

2. `makePendingOpenCodeProvider` signature changes from `(settings)` to `(settings, cwd)` — all callers must be updated. In our fork, the only callers are in `OpenCodeDriver.ts` lines 139-143 (`checkProvider`) and line 151 (`initialSnapshot`). The PR updates both.

## Detailed Code Review

### `parseSkillFrontmatter()` — YAML frontmatter parser

```typescript
function parseSkillFrontmatter(
  content: string,
): { name: string; description?: string; displayName?: string } | null {
```

**Concerns:**

1. **Fragile description regex**: The pattern `^description:\s*(?:>\s*\n([\s\S]*?)|(.+))?$` has issues:
   - The outer `?` makes the entire alternation optional — a line with `description:` followed by nothing will silently produce `undefined` instead of erroring or warning. This is probably intentional (optional description), but the regex complexity suggests more cases were intended.
   - `([\s\S]*?)` is lazy and will match only one character on the next line before stopping — this only works because `.match(/^---\n([\s\S]*?)\n---/)` extracted the frontmatter block, so the "next line" within that block is the continuation.
   - YAML folded scalars (`>`) can span multiple indented lines with leading whitespace. The regex only captures one extra line.
   - **Recommendation**: Use a proper YAML parser (`yaml` package already in deps?) or at minimum handle multi-line folded scalars correctly.

2. **No `|` (literal block scalar) support**: YAML literal blocks (`description: |`) are not handled. Many SKILL.md files use this format.

3. **No YAML comment handling**: Lines starting with `#` inside the frontmatter block are not stripped.

### `loadOpenCodeSkills()` — Directory scanner

```typescript
function loadOpenCodeSkills(
  cwd: string,
): Effect.Effect<ReadonlyArray<ServerProviderSkill>, never, FileSystem.FileSystem | Path.Path>;
```

**Good:**

- Graceful degradation: missing directory → empty array, unreadable files → skipped, invalid frontmatter → skipped
- Skills are marked `enabled: true` by default
- `path` and `description`/`displayName` are properly wired

**Concerns:**

1. **Non-idiomatic Effect error handling**: Uses `Effect.exit` + `_tag` checks for control flow. More idiomatic Effect would use `Effect.catchAll` or `Effect.orElseSucceed`:

   ```typescript
   // Current:
   const entriesExit = yield * Effect.exit(fs.readDirectory(skillsDir));
   if (entriesExit._tag !== "Success") {
     return [];
   }
   // Better:
   const entries =
     yield *
     fs
       .readDirectory(skillsDir)
       .pipe(Effect.catchAll(() => Effect.succeed([] as ReadonlyArray<string>)));
   ```

2. **`shortDescription` truncation**: Cuts at 200 chars regardless of word boundaries — could split mid-word. Should truncate at last space before limit.

3. **No validation of `name`**: Could contain characters that break the slash-command system (e.g., `/`, `$`, spaces). The Codex driver's skill names are validated server-side; local loading skips this.

4. **`recursive: false` on `readDirectory`**: Only reads one level deep. If skill directories are nested (unlikely but possible), they'd be missed. This matches the Codex driver behavior though.

### OpenCodeDriver.ts changes

```
+import * as Layer from "effect/Layer";
+import * as NodeFileSystem from "@effect/platform-node/NodeFileSystem";
```

The `Layer.merge(Path.layer, NodeFileSystem.layer)` provision is added to two call sites. This is correct — `Path.layer` is the default implementation and `NodeFileSystem.layer` provides the Node.js filesystem. Both are required by `loadOpenCodeSkills`.

**Concern**: `Path.layer` is provided alongside `NodeFileSystem.layer`, but `NodeFileSystem.layer` might already include a `Path` implementation. Providing both is harmless (the later one wins), but slightly redundant.

## Conflict Analysis with PR #2891

**PR #2891** ([pingdotgg/t3code#2891](https://github.com/pingdotgg/t3code/pull/2891)) loads OpenCode skills from the server API (`client.app.skills()`) instead of the local filesystem.

### Conflict Points

| Location                                     | PR #2811                                     | PR #2891                                                 | Resolution                          |
| -------------------------------------------- | -------------------------------------------- | -------------------------------------------------------- | ----------------------------------- |
| `OpenCodeProvider.ts` import                 | Adds `ServerProviderSkill`                   | Adds `ServerProviderSkill`                               | Trivial — keep one                  |
| `OpenCodeProvider.ts` import                 | Adds `FileSystem`, `Path`                    | Adds `OpenCodeSkill` type                                | Keep both                           |
| `checkOpenCodeProviderStatus` success path   | Adds `skills` from `loadOpenCodeSkills(cwd)` | Adds `skills` from `mapOpenCodeSkills(inventory.skills)` | **MERGE**: call both, merge results |
| `makePendingOpenCodeProvider`                | Adds `cwd` param + `loadOpenCodeSkills`      | No change                                                | PR #2811 wins                       |
| `opencodeRuntime.ts` `OpenCodeInventory`     | No change                                    | Adds `skills: OpenCodeSkill[]`                           | PR #2891 wins                       |
| `opencodeRuntime.ts` `loadOpenCodeInventory` | No change                                    | Adds `loadSkills` + concurrent call                      | PR #2891 wins                       |

### Recommended Merge Strategy

**Apply PR #2811 first, then merge PR #2891 on top.** The combined approach gives:

- **Offline/startup**: Skills from local `.agents/skills/` (PR #2811)
- **After probe**: Skills from opencode server API (PR #2891)

In `checkOpenCodeProviderStatus`, deduplicate by skill `name` (server skills should take precedence over local skills when both exist):

```typescript
const localSkills = yield * loadOpenCodeSkills(cwd);
const serverSkills = mapOpenCodeSkills(inventoryExit.value.skills);
const serverNames = new Set(serverSkills.map((s) => s.name));
const skills = [...serverSkills, ...localSkills.filter((s) => !serverNames.has(s.name))];
```

### Risk Assessment

- **Merge complexity**: Moderate. Both modify the same 2-4 line region in `checkOpenCodeProviderStatus` success path.
- **Behavioral conflict**: Low. The approaches are complementary (local vs server), not competing.
- **Test coverage**: Neither PR adds tests for the skills loading. This is a gap — a test verifying skills appear in provider snapshots would catch regressions from the merge.

## Overall Assessment

**PR #2811 is ready to apply to the fork with the noted adjustments.**

- The change is well-scoped, additive, and mirrors existing patterns
- Error handling is robust (graceful degradation on missing/unreadable files)
- The YAML parser has known limitations but covers the common case
- Conflicts with PR #2891 are manageable with the deduplication strategy above

**Recommendation**: Apply to fork, then merge PR #2891 on top with deduplication.
