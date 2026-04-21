# Code Quality Audit — Shugu-loop

**Date:** 2026-04-21  
**Scope:** `/tmp/shugu-loop/src/` — TypeScript files  
**Auditor:** Hermes Agent

---

## Summary

| Category | Severity | Count |
|----------|----------|-------|
| Function Complexity >30 lines | Medium | 5 |
| Magic Numbers | Low | 12 |
| Missing TypeScript Types (`any`) | Low | 1 |
| TODO/FIXME Comments | Medium | 5 |
| Empty Catch Blocks | Low | 3 |

---

## 1. Function Complexity >30 Lines

### 1.1 `runLoop()` — 745 lines (GOD OBJECT)

**File:** `src/engine/loop.ts:99-745`  
**Severity:** Medium  
**Impact:** Single function handling: streaming, tool execution, hook dispatch, budget tracking, interruption, reflection, and event emission. Hard to test, debug, and maintain.

**Note:** This appears intentional as the "main loop" — consider extracting into smaller generators/steps.

---

### 1.2 `PluginHost.start()` — ~130 lines

**File:** `src/plugins/host.ts:253-380`  
**Severity:** Medium  
**Impact:** Handles Docker detection, spawn strategy selection, child process setup, and listener attachment all in one function.

---

### 1.3 `extractWorkContext()` — ~100 lines

**File:** `src/context/session/work-context.ts:53-160`  
**Severity:** Low-Medium  
**Impact:** Single function scanning message history, matching tool uses with results, and building context.

---

### 1.4 `detectProject()` — ~80 lines

**File:** `src/commands/init.ts:35-110`  
**Severity:** Low  
**Impact:** Detects project structure, language, framework, and infrastructure in one function.

---

### 1.5 `MetaEvaluator.evaluate()` — ~60 lines

**File:** `src/meta/evaluator.ts:61-120`  
**Severity:** Low  
**Impact:** Manages task evaluation loop with budget tracking and result aggregation.

---

## 2. Magic Numbers

### 2.1 Timeout Values

| Value | File | Line | Context |
|-------|------|------|---------|
| `5000` | `src/plugins/host.ts` | 103 | Docker version check timeout |
| `5000` | `src/commands/doctor.ts` | 79, 88 | Git exec timeout |
| `5000` | `src/automation/daemon.ts` | 169 | Restart cooldown |
| `5000` | `src/automation/daemon.ts` | 217 | Force kill timeout |
| `1000` | `src/automation/daemon.ts` | 459 | Exit delay |
| `30000` | `src/plugins/host.ts` | 243 | Default plugin timeout |

**Suggested Fix:** Extract to named constants at the top of each file.

---

### 2.2 UID/GID Values

| Value | File | Line | Context |
|-------|------|------|---------|
| `65534` | `src/plugins/host.ts` | 324-325 | `nobody` user for Docker fallback |

**Suggested Fix:** Use named constant like `NOBODY_UID = 65534`.

---

### 2.3 Size Limits

| Value | File | Line | Context |
|-------|------|------|---------|
| `5000` | `src/skills/bundled/vibe.ts` | 352 | File content slice limit |
| `204800` | `src/engine/budget.ts` | 132 | Default context window |
| `80` | `src/context/session/work-context.ts` | 96 | Tool result summary length |

**Suggested Fix:** Extract to configuration constants.

---

## 3. Missing TypeScript Types (`any`)

### 3.1 `remote/gateway.ts` — Dynamic WebSocket import

**File:** `src/remote/gateway.ts:81`  
**Severity:** Low  
**Impact:** Uses `any` for `WebSocketServer` due to optional dependency pattern.

```typescript
let WebSocketServer: any;
try {
  const ws = await import('ws');
  WebSocketServer = ws.WebSocketServer ?? ws.default?.WebSocketServer;
} catch {
  throw new Error('WebSocket server requires the "ws" package...');
}
```

**Suggested Fix:** Define a proper interface for the ws module or use `@ts-expect-error` with a comment explaining the optional dependency.

---

## 4. TODO/FIXME Comments

### 4.1 `skills/generator.ts:53`

```typescript
// TODO: Implement the skill logic
return {
```

**Severity:** Medium  
**Impact:** Skeleton implementation — skill generation is not functional.

---

### 4.2 `meta/evaluator.ts:234`

```typescript
// TODO: implement with MiniMax call using rubric
return { score: 0.5, criteriaResults: [], success: true };
```

**Severity:** Medium  
**Impact:** LLM-as-judge scoring is stubbed to return 0.5.

---

### 4.3 `context/session/work-context.ts:160-162`

```typescript
/(?:remaining|restant|TODO|still need to)(.{10,100})/i,
/(?:step \d+ of \d+)(.{0,100})/i,
```

**Severity:** Low  
**Impact:** Pattern detection for "pending work" — patterns exist but may need refinement.

---

### 4.4 `automation/proactive.ts:8-10`

```typescript
 * Use cases:
 * - "Finish implementing all pending TODOs in this file"
 * - "Monitor the CI pipeline and fix failures"
```

**Severity:** Low  
**Impact:** Documentation comments describing planned features.

---

### 4.5 `plugins/builtin/behavior-hooks.ts:7-9`

```typescript
 * Hooks:
 * 1. Anti-laziness: detect TODO/stub/truncation in Write/Edit outputs
 * 2. Secret scanning: detect API keys/tokens in Bash output
```

**Severity:** Low  
**Impact:** Documentation describing behavior hook functionality.

---

## 5. Empty Catch Blocks

### 5.1 `context/memory/store.ts:135-137`

```typescript
} catch {
  // Skip unreadable files
}
```

**Severity:** Low  
**Impact:** Silently skips files that can't be read. May hide permission issues or corruption.

---

### 5.2 `commands/init.ts:65` (similar pattern)

```typescript
} catch { /* skip */ }
```

**Severity:** Low  
**Impact:** Skips directories that can't be stat'd during project detection.

---

### 5.3 `skills/bundled/vibe.ts:353`

```typescript
} catch {
  // Skip unreadable outputs
}
```

**Severity:** Low  
**Impact:** Silently skips output files that can't be read.

---

## 6. Code Duplication

### 6.1 Magic Number Pattern Repetition

The value `5000` appears in multiple files for different timeout purposes:
- `plugins/host.ts:103` — Docker version check
- `commands/doctor.ts:79,88` — Git commands
- `automation/daemon.ts:169,217` — Process management
- `skills/bundled/vibe.ts:352` — File reading

**Suggested Fix:** Centralize timeout constants in a shared config module.

---

### 6.2 Similar Hook Handler Patterns

**Files:** `plugins/hooks.ts:119-213`  
**Pattern:** Multiple `for...of` loops with `try/catch` calling handlers identically.

```typescript
for (const hook of handlers) {
  try {
    await hook.handler(...);
  } catch (err) {
    logger.debug(`Hook ${type} failed:`, err);
  }
}
```

**Suggested Fix:** Extract to a `runHooks()` utility function.

---

## 7. Recommendations

| Priority | Issue | File | Suggested Fix |
|----------|-------|------|---------------|
| High | TODO: unimplemented skill logic | `generator.ts:53` | Implement or remove stub |
| High | TODO: LLM-as-judge scoring | `evaluator.ts:234` | Implement rubric-based scoring |
| Medium | `runLoop()` complexity | `loop.ts:99` | Extract streaming/execution/budget into separate functions |
| Medium | Magic numbers scattered | Multiple | Create `config/timeouts.ts` |
| Low | Empty catch blocks | Multiple | Add logging for debugging |
| Low | `any` type for ws module | `gateway.ts:81` | Define interface for optional dep |

---

## 8. Positive Findings

1. **Well-structured types** — Most files use explicit interfaces and types
2. **Descriptive naming** — Functions and variables generally have clear names
3. **Good error messages** — Custom error classes in `transport/errors.ts`
4. **Security awareness** — `containsShellInjection()`, `buildSafeEnv()`, `timingSafeCompare()` patterns present

---

*Audit completed successfully. Total files analyzed: 50 TypeScript files.*
