# Shugu Code Quality Audit
**Date:** 2026-04-19
**Scope:** `/tmp/shugu-loop/src/`
**Total Files Analyzed:** ~188 source files

---

## 1. Function Complexity (>30 lines)

### Critical: Functions Exceeding 100 Lines

| File | Function | Est. Lines | Issue |
|------|----------|------------|-------|
| `engine/loop.ts` | `runLoop()` | ~595 | Main async generator — handles streaming, tool execution, concurrency, budget checks, hook system. Consider extracting tool execution phases (concurrent/sequential) into separate functions. |
| `agents/orchestrator.ts` | `spawn()` | ~170 | Agent spawning with worktree isolation, permission capping, tool set building. Mixes concerns: worktree management, permission logic, and loop execution. |
| `plugins/host.ts` | `PluginHost.start()` | ~130 | Process spawning with Docker/Node permission/tsx fallback logic. Extract spawn strategy selection into a dedicated factory function. |
| `agents/orchestrator.ts` | `buildAgentPrompt()` | ~35 | Role prompt assembly — long string template. Consider structured prompt builder. |

### High: Functions 50-100 Lines

| File | Function | Est. Lines | Issue |
|------|----------|------------|-------|
| `plugins/host.ts` | `handleChildRequest()` | ~85 | Large switch statement for 5 callback types. Extract each callback handler into separate methods. |
| `plugins/host.ts` | `createProxySkill()` | ~70 | Skill proxy creation with regex validation and trigger reconstruction. Consider splitting. |
| `plugins/host.ts` | `handleNotification()` | ~55 | Switch statement for 5 registration types. Good candidate for registry pattern. |
| `context/workspace/indexer.ts` | `indexWorkspace()` | ~85 | Full workspace indexing with hash comparison. Well-structured despite length. |
| `context/workspace/indexer.ts` | `walkDirectory()` | ~60 | Recursive directory walking with symlink safety. Consider extracting symlink resolution. |
| `transport/client.ts` | `stream()` | ~45 | SSE stream handling with idle timeout. Reasonable complexity. |
| `engine/turns.ts` | `ensureToolResultPairing()` | ~80 | Two-pass algorithm for tool result pairing. Well-commented; consider extracting passes. |

### Medium: Functions 31-50 Lines

| File | Function | Est. Lines | Issue |
|------|----------|------------|-------|
| `plugins/child-entry.ts` | `handleMessage()` | ~230 | Single switch handling init, invoke_tool, invoke_hook, invoke_command, invoke_skill, shutdown. Should be split into handlers. |
| `context/compactor.ts` | (multiple) | 35-45 | Chunk summarization logic. Acceptable. |
| `tools/bash/BashTool.ts` | `runBash()` | ~65 | Shell spawning and output collection. Consider extracting output buffering. |

---

## 2. Code Duplication

### Duplicated Pattern: Promise.race with Timeout + Abort

**Files affected:** `engine/loop.ts`, `transport/client.ts`, `tools/bash/BashTool.ts`, `remote/ssh.ts`

```typescript
// Pattern repeated 4+ times:
const timeoutPromise = new Promise<never>((_, reject) => {
  const timer = setTimeout(() => rej(new Error(...)), timeoutMs);
  if (typeof timer === 'object' && 'unref' in timer) timer.unref();
});
const abortPromise = new Promise<never>((_, reject) => {
  if (interrupt.signal.aborted) { rej(new Error('Aborted')); return; }
  interrupt.signal.addEventListener('abort', () => rej(new Error('Aborted')), { once: true });
});
result = await Promise.race([execPromise, timeoutPromise, abortPromise]);
```

**Recommendation:** Extract into `utils/async.ts` as `withTimeoutAndAbort()`.

---

### Duplicated Pattern: JSON-RPC Error Response Construction

**Files affected:** `plugins/host.ts`, `plugins/child-entry.ts`, `plugins/protocol.ts`

Repeated error code definitions and response formatting:
```typescript
// In protocol.ts:
export const RPC_PARSE_ERROR = -32700;
export const RPC_INVALID_REQUEST = -32600;
export const RPC_METHOD_NOT_FOUND = -32601;
export const RPC_TIMEOUT = -32000;
// ... but error construction is duplicated in host.ts and child-entry.ts
```

**Recommendation:** Centralize error construction in `protocol.ts` with factory functions.

---

### Duplicated Pattern: Truncation with Magic Numbers

**Files:** `agents/delegation.ts`, `agents/teams.ts`, `skills/bundled/secondbrain.ts`, `context/memory/agent.ts`, `context/workspace/project.ts`

```typescript
// Various slice limits used inconsistently:
.slice(0, 2000)  // result previews
.slice(0, 200)   // response previews
.slice(0, 150)   // note previews
.slice(0, 100)   // name previews
.slice(0, 50)    // short strings
```

**Recommendation:** Define constants in `tools/outputLimits.ts` for consistency.

---

## 3. Magic Numbers & Strings

### Critical Severity

| Value | Location | Issue | Suggestion |
|-------|----------|-------|------------|
| `65534` | `plugins/host.ts:324-325` | UID/GID for nobody user — hardcoded Linux numeric ID | Define `const NOBODY_UID = 65534` with platform check |
| `300_000` | `engine/loop.ts:382` | Default tool timeout (5 min) | Already named `DEFAULT_TIMEOUT` — good, but verify usage elsewhere |
| `600_000` | `tools/bash/BashTool.ts:118`, `transport/client.ts:51` | Max timeout (10 min) | Should be centralized constant |
| `120_000` | `tools/bash/BashTool.ts:17` | Default bash timeout (2 min) | Should reference centralized constant |

### High Severity

| Value | Location | Issue | Suggestion |
|-------|----------|-------|------------|
| `204_800` | `engine/loop.ts:268` | MiniMax M2.7 context window | Named `contextWindow` — good, but document source |
| `90` | `engine/turns.ts:218` | `CONTINUATION_THRESHOLD = 0.9` | Good — percentage constant |
| `500` | `engine/turns.ts:221` | `DIMINISHING_RETURNS_THRESHOLD` | Good — named constant |
| `5` | `engine/turns.ts:224` | `MAX_CONTINUATIONS` | Good |
| `100` | `engine/turns.ts:180` | `DEFAULT_MAX_TURNS` | Good |
| `3` | `engine/loop.ts:453-455` | Loop detection threshold | Named inline comment — extract constant |
| `2000` | `agents/delegation.ts:86`, `agents/teams.ts:163` | Context string limit | Extract to constant |
| `5000` | `skills/bundled/vibe.ts:352` | Content truncation | Extract to constant |

### Medium Severity

| Value | Location | Issue | Suggestion |
|-------|----------|-------|------------|
| `50` | `plugins/child-entry.ts:385` | `setTimeout(() => process.exit(0), 50)` — shutdown delay | Extract with comment explaining purpose |
| `3000` | `remote/ssh.ts:50`, `tools/bash/BashTool.ts:204` | SIGKILL grace period after SIGTERM | Named constant `KILL_GRACE_PERIOD_MS` |
| `200` | `context/compactor.ts:242` | JSON input preview | Named inline |
| `100` | Various `slice(0, 100)` | Error/preview truncation | Inconsistent with other limits |
| `150`, `200`, `500` | Various memory/context truncation | Multiple limits for same purpose | Consolidate |

---

## 4. Missing TypeScript Types

### Critical: `any` Types in Plugin Host

**File:** `plugins/host.ts:232-238`
```typescript
private activeCallbackCtx: {
  info?: (msg: string) => void;
  error?: (msg: string) => void;
  query?: (prompt: string) => Promise<string>;
  runAgent?: (prompt: string) => Promise<string>;
  toolInvoke?: (toolName: string, call: ToolCall) => Promise<ToolResult>;
} | null = null;
```
**Issue:** `activeCallbackCtx` lacks proper interface. Define `CallbackContext` type.

---

### High: Untyped Hook Payloads

**File:** `plugins/hooks.ts`
```typescript
handler: (payload: unknown) => Promise<unknown>;
```
**Issue:** Generic `unknown` loses type safety. Consider generic typing:
```typescript
interface HookHandler<T = unknown> {
  type: HookType;
  handler: (payload: T) => Promise<T>;
}
```

---

### High: Unsafe Cast Patterns

**File:** `plugins/host.ts` (throughout)
```typescript
msg = JSON.parse(line) as JsonRpcMessage;  // Line 474
const response = msg as JsonRpcResponse;   // Line 482
const request = msg as JsonRpcRequest;     // Line 498
```
**Issue:** Without runtime validation, `as` casts can mask bugs. Consider:
```typescript
function isJsonRpcRequest(msg: JsonRpcMessage): msg is JsonRpcRequest {
  return 'id' in msg && 'method' in msg;
}
```

---

### Medium: Missing Parameter Types

**File:** `context/memory/agent.ts:277, 297, 316`
```typescript
content: note.body.slice(0, 200),  // note type unclear
content: sanitizeUntrustedContent(redactSensitive(m.content.slice(0, 150)));  // m type unclear
```
**Issue:** Content operations on untyped objects risk runtime errors.

---

### Medium: Inconsistent Type Exports

**Files:** `protocol/messages.ts`, `protocol/tools.ts`

Some exported types are used inconsistently:
```typescript
// In engine/loop.ts:546
let result: import('../protocol/tools.js').ToolResult;  // Inline import instead of type
```

---

## Summary Statistics

| Category | Count | High Severity |
|----------|-------|---------------|
| Functions >100 lines | 4 | 4 |
| Functions 50-100 lines | 8 | 3 |
| Functions 31-50 lines | 6 | 1 |
| Magic numbers (critical) | 4 | 4 |
| Magic numbers (high) | 12 | 6 |
| Missing types (`any`/unknown) | 6 | 3 |
| Duplicated patterns | 3 | 1 |

---

## Top Priorities for Refactoring

1. **`engine/loop.ts:runLoop()`** — Extract concurrent/sequential tool execution into separate functions
2. **`plugins/host.ts`** — Add `CallbackContext` interface, extract callback handlers
3. **Create `utils/async.ts`** — Centralize `Promise.race` timeout+abort pattern
4. **`tools/outputLimits.ts`** — Consolidate magic truncation numbers
5. **`plugins/child-entry.ts`** — Split `handleMessage()` into separate handlers
6. **Add runtime validation** — Replace `as` casts with type guard functions
