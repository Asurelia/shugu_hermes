# [P3] — Performance: EventEmitter subclasses lack listener cleanup

**File:** Multiple
**Line:** Various
**Severity:** P3 (Low)
**Category:** Performance

## Description

Multiple EventEmitter subclasses never call `removeListener()` or `removeAllListeners()`. While many are long-lived singletons where this is acceptable, some cases (e.g., per-session or per-task event listeners) could accumulate over time.

## Evidence

EventEmitter usage without cleanup:
- `src/utils/tracer.ts:66` — `_emitter` (singleton, never cleaned)
- `src/automation/background.ts:53` — `BackgroundManager`
- `src/automation/scheduler.ts:144` — `Scheduler`
- `src/automation/triggers.ts:65` — `TriggerServer`
- `src/automation/daemon.ts:72` — `DaemonController`
- `src/automation/daemon.ts:359` — `DaemonWorker`
- `src/automation/proactive.ts:82` — `ProactiveLoop`
- `src/skills/loader.ts:94` — `SkillRegistry`
- `src/plugins/registry.ts:19` — `PluginRegistry`
- `src/plugins/hooks.ts:90` — `HookRegistry`
- `src/plugins/host.ts:206` — `PluginHost`

No calls to `removeListener`, `removeAllListeners`, or `off` found anywhere in the codebase.

## Impact

- Node.js emits a console warning when a single emitter has 10+ listeners
- Per-session/proactive-loop emitters that add/remove many listeners without cleanup → memory leak
- Very long CLI sessions with many proactive loops could accumulate listeners

## Recommended Fix

For short-lived emitters (BackgroundManager, DaemonWorker, ProactiveLoop), call `removeAllListeners()` when done, or use `{ once: true }` options for one-time listeners.

For long-lived singletons (tracer, registries), this is acceptable.
