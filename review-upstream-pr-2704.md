# Review: Upstream PR #2704 — Fix OpenCode Event Ingestion

**PR**: https://github.com/pingdotgg/t3code/pull/2704
**Author**: eersnington (Sree Narayanan)
**Status**: Open
**Files**: `OpenCodeAdapter.ts` (+99/−7), `OpenCodeAdapter.test.ts` (+249/−7)

## Summary

OpenCode CLI >= 1.14.40 changed its event format: SSE events are now wrapped in
a `{ directory, payload }` envelope via a new `/global/event` endpoint. The
existing adapter calls `event.subscribe()` which returns raw events — but
`handleSubscribedEvent` reads `event.properties.sessionID` directly, which on
the wrapped payload is `event.payload.properties.sessionID`. This mismatch
causes ALL events from newer OpenCode versions to be silently discarded.

This PR fixes event ingestion by:
1. Preferring `global.event` — the new endpoint that returns directory-scoped
   wrapped events
2. Filtering events by session `directory` so multi-session servers don't
   cross-contaminate
3. Validating payload shape (`parseOpenCodeSubscribedEvent`) before projection,
   squashing malformed events instead of crashing
4. Falling back to legacy `event.subscribe({ directory })` when `global.event`
   is unavailable (404 or `TypeError: global.event is not a function`)

## Root Cause Analysis

From issue #2691: newer OpenCode server wraps events as:
```json
{ "directory": "/path/to/project", "payload": { "type": "session.status", "properties": {...} } }
```

The existing `handleSubscribedEvent` at line ~311 reads `event.properties`
directly. On wrapped events this is always `undefined` → every event silently
fails the session ID check and is discarded. The session stays in "running"
state indefinitely (issue #2644), no content renders (issue #2652), and the
adapter appears "stuck" when the underlying OpenCode session has already
completed.

## Issue Assessment

| Issue | Fixed? | Evidence |
|-------|--------|----------|
| #2644 — "working..." hangs indefinitely | **Yes** | global.event path delivers properly unwrapped events; session.idle status triggers `turn.completed` |
| #2652 — messages save to opencode.db but never render | **Yes** | Same root cause; payload.unwrap → content.delta events reach the UI |
| #2691 — SSE events silently dropped | **Yes** | `parseOpenCodeSubscribedEvent` validates payload shape; global path unwraps envelope; directory filter prevents cross-session discard |

The PR title says "Fixes #2644" but the root cause is shared across all three
issues. The directory filtering + payload validation in the PR handles both the
envelope mismatch AND a secondary failure mode where malformed events from
other sessions could crash the pump.

## Fork Compatibility

### Merge Risk: LOW-MEDIUM

The PR touches only `OpenCodeAdapter.ts` (event pump rewrite) and its tests.
These are additive changes — new interfaces, new helpers, new logic branches.
The legacy `event.subscribe` path is retained as the fallback.

**Potential conflicts with fork divergences:**

| Fork commit | Risk | Notes |
|-------------|------|-------|
| #2840 — Effect beta.73 migration | **Low** | `OpenCodeRuntimeError` still uses `Data.TaggedError` in fork; PR doesn't touch error type |
| #2526 — Fix OpenCode raw text delta assembly | **Low** | That change is in `handleSubscribedEvent`, which the PR passes through unchanged |
| #2596 — Stricter Effect LSP rules | **None** | Lint-only; no logic changes |

**SDK compatibility:** Fork uses `@opencode-ai/sdk@^1.3.15` (installed 1.3.15).
The `global.event` endpoint may not exist in the `OpencodeClient` type at this
version. The PR handles this gracefully:
- `OpenCodeGlobalEventClient` uses optional chaining (`global?.event?.()`)
- `runOpenCodeSdk` wraps the call in try/catch → `TypeError` on missing
  method triggers the fallback to `event.subscribe`
- Users on older SDKs get the legacy path automatically; upgrading the SDK
  unlocks the new path

**No change to `opencodeRuntime.ts` needed.** The PR adds all logic within
`OpenCodeAdapter.ts`. The `createOpenCodeSdkClient` return type (`OpencodeClient`)
already has the shape the PR casts to.

## Code Quality

**Strengths:**
- Clean separation: `runGlobalEvents` + `runLegacyEvents` are distinct, readable paths
- `parseOpenCodeSubscribedEvent` with runtime validation is defensive against
  malformed server responses — won't crash the pump
- Directory filtering prevents multi-session servers from leaking events
- `isOpenCodeGlobalEventUnavailable` handles both 404 (no endpoint) and
  TypeError (no method on client) — complete fallback coverage
- Tests cover all four branches: global events, directory filtering, malformed
  payloads, and fallback path

**Concerns:**
- `OpenCodeGlobalEventClient` is a manual type cast of `OpencodeClient`; if the
  SDK adds `global.event` with a different API shape, type-checking won't catch it.
  The `runOpenCodeSdk` try/catch mitigates runtime impact.
- `openCodeRuntimeErrorStatus` chains through `cause.cause.response.status` —
  fragile to error-wrapping changes. However, the current error chain in the
  fork is stable.

## Recommendation

**Apply.** This PR fixes a critical bug affecting ALL OpenCode users on CLI
versions >= 1.14.40 (the current latest). Without it, the OpenCode adapter is
functionally broken — events arrive but are silently discarded. The PR is
well-structured, thoroughly tested, and falls back gracefully on older SDKs.

**Merge priority:** Cherry-pick the event pump rewrite and helpers together as
one unit. The test changes can come separately if the fork's test setup has
diverged.

**Merge order note:** If applying PR #2887 (Schema.TaggedErrorClass migration)
first, re-check `openCodeRuntimeErrorStatus` which accesses `cause.cause`.
The `Schema.TaggedErrorClass` changes internal error structure slightly
(`.make()` vs `new`, but the `cause` field semantics are preserved).

**If SDK upgrade is needed:** Bump `@opencode-ai/sdk` to >= 1.4.7 (the
version repo'd with this fix) to get native `global.event` endpoint support.
Without the bump, the legacy fallback path still works — the fix is the
directory-aware `event.subscribe({ directory })` call, which works on any
SDK version.

## Diff Summary by Change

| Change | Lines | Impact |
|--------|-------|--------|
| `OpenCodeGlobalEventEnvelope` + `OpenCodeGlobalEventClient` types | +5 | Additive — no existing code touched |
| `parseOpenCodeSubscribedEvent()` | +17 | New runtime validation — defensive |
| `openCodeRuntimeErrorStatus()` + `isOpenCodeGlobalEventUnavailable()` | +22 | New helper to detect fallback conditions |
| `startEventPump` rewrite (global → legacy) | +55/−11 | Core fix — replaces flat `event.subscribe` with two-tier approach |
| Test mock: global.event + keepSubscriptionOpen | +32 | New mock capabilities for test harness |
| Tests: 4 new cases | +217 | Covers all branches |
