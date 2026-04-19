# [P3] — Architecture Audit

**Date:** 2026-04-19
**Severity:** P3 (Enhancement / Low)

---

## Summary

| Category | Finding | Severity |
|----------|---------|----------|
| God Objects | `engine/loop.ts:runLoop()` at ~595 lines | P3 |
| Unclear Boundaries | Plugin child `handleMessage()` at ~230 lines | P3 |
| Missing Error Boundaries | Plugin child-to-parent error propagation | P3 |
| Unused Field | `resolvedPermissions` in `PluginHostOptions` | P4 |

---

## 1. God Objects / Complex Functions

### `runLoop()` — 595 lines

**File:** `src/engine/loop.ts`

The main async generator handles:
- Streaming setup and teardown
- Tool execution (concurrent + sequential phases)
- Budget checks and loop detection
- Hook system execution
- Message building
- Abort signal handling

**Issue:** Single responsibility principle violated. Changes to tool execution logic risk breaking streaming, and vice versa.

**Recommended Fix:** Extract phases into separate generators or handler classes.

---

## 2. Plugin Child `handleMessage()` — 230-line switch

**File:** `src/plugins/child-entry.ts`

Single `handleMessage()` method with a switch handling: init, invoke_tool, invoke_hook, invoke_command, invoke_skill, shutdown.

**Recommended Fix:** Extract each message type into a separate handler method on the class.

---

## 3. Plugin Error Boundaries

**File:** `src/plugins/child-entry.ts:408`

```typescript
handleMessage(msg).catch(...)
```

Errors in `handleMessage()` are caught at the top level but the error propagation path back to the host parent is unclear. Malformed stdin JSON is written to child's stderr only (see error-handling audit finding #4).

**Recommended Fix:** Ensure structured error responses are sent back to the host for all error types, not just certain subclasses.

---

## 4. Unused Field

**File:** `src/plugins/host.ts:76`

```typescript
export interface PluginHostOptions {
  // ...
  resolvedPermissions?: string[];
}
```

`resolvedPermissions` exists but is never used. The actual permission gating happens via `hasPermission()` reading `manifest.permissions` directly.

**Recommended Fix:** Either wire it up or remove it.

---

## 5. Circular Dependencies (Not Found)

No circular dependencies detected in the import graph during this audit.

---

## 6. Positive Findings

| Aspect | Finding |
|--------|---------|
| Modular design | Clear separation: transport, protocol, policy, plugins, agents |
| Error escalation | `plugin-host.ts:475–477` properly emits error events for malformed JSON |
| Capability broker | Strong isolation for brokered plugins with path validation |
| Transport abstraction | Clean separation between SSE and WebSocket transports |
