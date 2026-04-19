# [P3] — Code Quality: Complex functions needing refactoring

**Date:** 2026-04-19
**Severity:** P3

## Description

18 high-complexity functions identified across the codebase. Top priorities:

### Critical (>100 lines)
| File | Function | Lines |
|------|----------|-------|
| `engine/loop.ts` | `runLoop()` | ~595 |
| `agents/orchestrator.ts` | `spawn()` | ~170 |
| `plugins/host.ts` | `PluginHost.start()` | ~130 |

### High (50–100 lines)
| File | Function | Lines |
|------|----------|-------|
| `plugins/host.ts` | `handleChildRequest()` | ~85 |
| `plugins/host.ts` | `createProxySkill()` | ~70 |
| `context/workspace/indexer.ts` | `indexWorkspace()` | ~85 |

## Recommended Fixes

1. **`runLoop()`** — Extract concurrent/sequential tool execution into separate generator functions
2. **`handleChildRequest()`** — Extract each callback handler into separate named methods
3. **`handleMessage()` in `child-entry.ts`** — Split the 230-line switch into per-method handlers
4. **Create `utils/async.ts`** — Centralize `Promise.race` timeout+abort pattern (used 4+ times)
5. **Create `tools/outputLimits.ts`** — Consolidate magic truncation numbers
6. **Add `CallbackContext` interface** in `plugins/host.ts` for the untyped `activeCallbackCtx`

## Duplicated Patterns

- `Promise.race` timeout+abort: `engine/loop.ts`, `transport/client.ts`, `tools/bash/BashTool.ts`, `remote/ssh.ts`
- JSON-RPC error construction: `plugins/host.ts`, `plugins/child-entry.ts`, `protocol.ts`
- Truncation with magic numbers: `agents/delegation.ts`, `agents/teams.ts`, `skills/bundled/secondbrain.ts`
