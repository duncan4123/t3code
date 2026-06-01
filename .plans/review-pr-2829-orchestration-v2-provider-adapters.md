# Review: PR #2829 — WIP wire orchestration v2 provider adapters

**Date:** 2026-06-01
**Reviewer:** gastown.furiosa (t3code polecat)
**Bead:** t3-29f.6
**PR:** https://github.com/pingdotgg/t3code/pull/2829
**Branch:** `t3code/codex-turn-mapping` (draft, not merged)
**Author:** juliusmarminge
**Scope:** +43,926 / −18 lines, 171 files
**State:** DRAFT (open)

## PR Summary

Introduces a comprehensive orchestration v2 runtime under `apps/server/src/orchestration-v2/`:
- **Provider adapter layer**: Codex and Claude adapter drivers (`CodexAdapterV2.ts`, `ClaudeAdapterV2.ts`) with capability matrices, protocol logging, and a provider adapter registry/factory.
- **Domain event store**: Event sourcing with `EventStore`, `EventSink`, `ProjectionStore` for thread, run, run attempt, node, and provider session projections.
- **Session management**: `ProviderSessionManager` for provider lifecycle (start, stop, reconnect), `CheckpointService` for session snapshots, `ContextHandoffService` for cross-session context transfer.
- **Run orchestration**: `Orchestrator` for dispatch/execution lifecycle, `RunExecutionService` for run state transitions, `CommandPolicy` for capability-based authorization.
- **Thread operations**: `ThreadForkService` for fork/rollback, lineage tracking, and merge-back handoff support.
- **Database migration**: `031_OrchestrationV2.ts` creates orchestration v2 event, command receipt, and projection tables.
- **Replay testkit**: Deterministic replay with provider-specific harnesses, NDJSON transcript fixtures, ~20 scenario fixtures (simple, multi-turn, tool calls, interrupts, thread forks, rollbacks, web search, subagents, plan questions).
- **WebSocket RPCs**: `dispatchCommand`, `getThreadProjection`, `subscribeShell`, `subscribeThread` exposed through server WS layer.
- **Shared contracts**: Strongly-typed schemas in `orchestrationV2.ts` for commands, domain events, projections, stream items, and RPC schemas.

## Relevance Assessment

### #2838 — OpenCode (ACP) provider creates new session instead of resuming

**Relevance: HIGH — architectural fix**

The root cause is that the current T3 Code provider layer does not persist ACP session IDs or call `resumeSession` on reconnect. The orchestration v2 `ProviderSessionManager` directly addresses this:
- Tracks session state through the `ProjectionStore` (active, reaped, stopped)
- Persists provider session IDs as first-class domain entities
- `CheckpointService` snapshots session state for crash recovery
- `ContextHandoffService` manages context transfer on session resume
- `ProviderAdapter` interface defines `startSession` vs `resumeSession` as distinct operations

When an OpenCode adapter is wired into this framework, the session resume path would be handled by the projection store automatically — the provider session ID is persisted in the projection, and on reconnect the orchestrator loads the projection and calls resume instead of start.

**Gap:** The PR currently has Codex and Claude adapters but no OpenCode/ACP adapter yet. The OpenCode provider would need its own `OpenCodeAdapterV2` wired into the `ProviderAdapterRegistry`.

### #2778 — Session hung forever after spawning subagents

**Relevance: MEDIUM — infrastructure, not direct fix**

Root cause: child subagent session emits `permission.asked` events for `external_directory` that never reach the T3 Code UI. The v2 architecture provides infrastructure that could fix this:
- `ProviderEventIngestor` forwards provider events into the domain event stream
- `ProjectionStore` tracks session state including child sessions
- Subagent scenario fixtures exist in `testkit/fixtures/subagent/`

However, the permission-forwarding pipeline from provider → UI is not explicitly addressed in this PR. The `CommandPolicy` handles capability authorization but doesn't add new permission event routing. The fix for #2778 likely requires:
1. Ensuring child session permission events are ingested into the event store
2. Surfacing those events through the WebSocket RPC layer
3. The UI displaying permission prompts for child sessions

This PR provides the event infrastructure to make that possible, but doesn't close the loop on its own.

### #2886 — Thread stuck on status working with OpenCode as harness

**Relevance: HIGH — structural fix**

Root cause: thread state is managed with mutable flags that fail to transition when the provider-side run completes. The v2 architecture eliminates this pattern:
- Event sourcing ensures thread state is derived from domain events, not mutable flags
- `RunExecutionService` manages explicit run lifecycle states (started → completed/failed/cancelled)
- `Orchestrator` handles run completion through the event store, making state transitions atomic and recoverable
- The `DeterministicRuntime` replay testkit validates correct state transitions
- Mid-run message queuing is handled through `CommandReceiptStore`

If a provider run completes, the completion event is stored and projected — sticky "working" states cannot occur because the projection is always rebuilt from events. Even after app restart, the projection replays stored events and recovers the correct state.

**Gap:** This requires the OpenCode provider to correctly emit run-completion events, which means it needs to either use the V2 adapter interface or have its events bridged into the V2 event pipeline.

## Cross-Cutting Observations

### Risk
- **Migration 31** adds production tables (events, command receipts, projections). The PR README notes the V2 layer is "live alongside the existing v1 orchestration path" — both run concurrently. This is safe for deployment but increases database footprint.
- **Draft state**: The PR has 34 commits and hasn't been reviewed/merged. The TODO.md lists remaining work: "projection, context transfer, rollback, capability, and subagent work" — the PR is a snapshot of work in progress.

### Fork Compatibility
The `duncan4123/t3code` fork would need this PR merged and then the OpenCode ACP adapter added/bridged. The V2 architecture is provider-agnostic (the `ProviderAdapter` interface and `ProviderAdapterRegistry` are designed for multiple providers), so adding OpenCode support is a matter of implementing a new adapter driver.

### Conflict Check
This PR touches `apps/server/src/orchestration-v2/` exclusively (all new files). It does not modify existing orchestration v1 paths. The only cross-cutting change is `apps/server/package.json` (+2 lines) and the database migration. Risk of conflicts with other upstream PRs in the epic is low since each PR targets different areas.

## Recommendation

**This PR is directly relevant to all three issues.** The orchestration v2 architecture addresses the structural root causes:

| Issue | Root Cause | V2 Fix |
|-------|-----------|--------|
| #2838 | No ACP session ID persistence | ProviderSessionManager + ProjectionStore persist session IDs |
| #2778 | Permission events not forwarded | ProviderEventIngestor + event pipeline (infrastructure only — needs UI work) |
| #2886 | Mutable thread state flags | Event-sourced projections eliminate sticky states |

**Action:** Watch this PR for merge and plan an OpenCode ACP adapter V2 implementation on top of it. The PR itself does not fix the issues for OpenCode (since OpenCode has no V2 adapter yet), but it provides the architecture that makes the fixes straightforward.
