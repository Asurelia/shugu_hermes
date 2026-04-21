# Architecture Audit — Shugu-loop

**Date:** 2026-04-21  
**Scope:** `/tmp/shugu-loop/src/` — TypeScript files  
**Auditor:** Hermes Agent

---

## Summary

| Category | Severity | Count |
|----------|----------|-------|
| God Objects | Medium | 4 |
| Circular Dependencies | None | 0 |
| Missing Error Boundaries | High | 3 |
| Large Files (>400 LOC) | Medium | 7 |
| Missing Layer Boundaries | Low | 2 |

---

## 1. God Objects

### 1.1 `runLoop()` — 745 lines

**File:** `src/engine/loop.ts:99-745`  
**Severity:** Medium  
**Responsibilities:**
- Stream parsing and event emission
- Tool execution and hook dispatch
- Budget tracking and interruption
- Reflection injection
- Message history management
- Agent sub-loop spawning

**Concern:** Single function handles the entire agent lifecycle. If any of these responsibilities change, the entire function must be modified.

---

### 1.2 `PluginHost` — 858 lines

**File:** `src/plugins/host.ts:206-858`  
**Severity:** Medium  
**Responsibilities:**
- Child process spawning (Docker, tsx, Node --permission)
- IPC message handling (JSON-RPC)
- Capability brokering
- Tool/hook/command/skill registration proxying
- Lifecycle management (start, shutdown, kill)
- Listener cleanup

**Concern:** 650+ line class handling plugin isolation, security, and IPC. Multiple responsibilities should be separated.

---

### 1.3 `DaemonController` — 465 lines

**File:** `src/automation/daemon.ts:72-465`  
**Severity:** Medium  
**Responsibilities:**
- Process spawning and monitoring
- IPC server management (Unix socket/named pipe)
- State persistence (PID, logs)
- Auto-restart logic
- Signal handling

**Concern:** Combines daemon lifecycle with IPC server management.

---

### 1.4 `MetaEvaluator` — 431 lines

**File:** `src/meta/evaluator.ts:51-431`  
**Severity:** Medium  
**Responsibilities:**
- Task evaluation orchestration
- Git worktree management
- Score aggregation
- Result archival

---

## 2. Circular Dependencies

### 2.1 No Circular Dependencies Detected

**Analysis:** Searched for mutual imports between modules. The codebase uses a clear layer separation:

- **Layer 1 (Transport):** `MiniMaxClient`, `stream.ts`, `errors.ts` — no downward dependencies
- **Layer 2 (Engine):** `loop.ts`, `turns.ts`, `budget.ts` — imports from Layer 1
- **Layers 5-9 (Context/Agents/Automation):** Import from Engine and Protocol
- **Layer 14 (Plugins):** Isolated layer with `PluginHost` and `CapabilityBroker`

**Note:** `plugins/loader.ts` imports `PluginHost` from `plugins/host.ts` and `CapabilityBroker` from `plugins/broker.ts` — both within the same layer. No cross-layer circular deps found.

---

## 3. Missing Error Boundaries

### 3.1 Main Loop — No Try/Catch

**File:** `src/engine/loop.ts:99+`  
**Severity:** High  
**Impact:** If an unhandled exception occurs during tool execution or hook dispatch, the entire agent loop crashes without cleanup.

**Current:** The loop uses `try/catch` internally for specific operations but has no top-level error boundary.

---

### 3.2 REPL Entry Point — No Error Boundary

**File:** `src/entrypoints/repl.ts:354+`  
**Severity:** High  
**Impact:** Unhandled exceptions in the main REPL loop leave the terminal in an inconsistent state.

**Current:** `gracefulShutdown()` handles SIGINT/SIGTERM but not uncaught exceptions.

---

### 3.3 Tool Execution — No Error Boundary

**File:** `src/tools/executor.ts`  
**Severity:** High  
**Impact:** If a tool throws an exception, error propagation depends on individual tool implementations.

**Note:** The codebase uses `Result<T>` patterns in some places, but tool execution lacks consistent error boundary patterns.

---

## 4. Large Files (>400 LOC)

| File | Lines | Primary Concern |
|------|-------|-----------------|
| `plugins/host.ts` | 858 | Plugin lifecycle + IPC |
| `engine/loop.ts` | 745 | Main agent loop |
| `entrypoints/repl.ts` | 729 | REPL UI + command handling |
| `utils/tracer.ts` | 665 | Tracing infrastructure |
| `meta/cli.ts` | 640 | CLI argument parsing |
| `agents/orchestrator.ts` | 626 | Sub-agent management |
| `entrypoints/bootstrap.ts` | 618 | Initialization |

**Concern:** Large files increase cognitive load and make parallel development difficult.

---

## 5. Missing Layer Boundaries

### 5.1 Engine Imports UI Components

**File:** `src/engine/loop.ts:68`  
```typescript
buddyObserver?: import('../ui/companion/observer.js').BuddyObserver;
```

**Concern:** Engine layer (Layer 2) should not depend on UI layer (Layer 11). This creates coupling between testing environments.

**Suggested Fix:** Use inversion of control — pass `buddyObserver` as a generic `Observer` interface.

---

### 5.2 Tools Import From Multiple Layers

**Files:** `src/tools/*.ts`  
**Concern:** Tool implementations import from engine, transport, and protocol layers directly, creating tight coupling.

**Suggested Fix:** Tools should only depend on the protocol layer (types/interfaces).

---

## 6. Architectural Patterns

### 6.1 Good: Capability Broker Pattern

**File:** `src/plugins/broker.ts`  
**Positive:** Plugins don't access system resources directly — all access goes through the brokered capability system.

---

### 6.2 Good: EventEmitter for Hooks

**File:** `src/plugins/hooks.ts`  
**Positive:** Uses `EventEmitter` for hook dispatch, allowing dynamic registration.

---

### 6.3 Good: AsyncGenerator for Stream Events

**File:** `src/engine/loop.ts:99`  
**Positive:** `runLoop()` is an `AsyncGenerator` yielding events — allows UI to observe without coupling.

---

### 6.4 Good: Worktree Isolation for Meta-evaluation

**Files:** `src/agents/worktree.ts`, `src/meta/evaluator.ts`  
**Positive:** Each evaluation task runs in an isolated git worktree.

---

## 7. Recommendations

| Priority | Issue | File | Suggested Fix |
|----------|-------|------|---------------|
| High | No error boundary in runLoop | `loop.ts:99` | Wrap main loop in try/catch with error emission |
| High | No error boundary in REPL | `repl.ts:354` | Add top-level error handler |
| Medium | Engine imports UI | `loop.ts:68` | Use `Observer` interface |
| Medium | PluginHost god object | `host.ts:206` | Extract IPC and capability handling |
| Medium | Large files | Multiple | Extract private methods to separate modules |
| Low | Tool coupling | `tools/*.ts` | Introduce tool interface layer |

---

## 8. Observations

### 8.1 Plugin Isolation Architecture

The plugin system (`PluginHost` + `CapabilityBroker`) provides strong isolation:
- Plugins run in separate processes (Docker or Node --permission)
- All capabilities are explicitly brokered
- Tool/hook/command/skill registration is tightly controlled

**This is a well-designed security boundary.**

---

### 8.2 Layered Architecture

The codebase follows a layered architecture:

```
Layer 1:  Transport (HTTP, streaming, errors)
Layer 2:  Engine (loop, turns, budget)
Layer 3:  Protocol (messages, tools, actions)
Layer 5:  Context (memory, session, work-context)
Layer 6:  Skills (loader, generator, bundled)
Layer 7:  Policy (rules, security)
Layer 8:  Agents (orchestrator, worktree)
Layer 9:  Automation (daemon, scheduler, triggers)
Layer 10: Credentials (prompt, auth)
Layer 11: UI (render, highlight, statusbar, companion)
Layer 12: Commands (registry, init, doctor, automation)
Layer 13: Meta (evaluator, harness, runtime)
Layer 14: Plugins (host, loader, broker, hooks)
Layer 15: Remote (gateway)
```

**This is well-organized and maintainable.**

---

## 9. Security Architecture

### 9.1 Strengths

1. **Plugin sandboxing** via Docker or Node --permission flags
2. **Capability broker** for explicit resource access control
3. **Safe env** sanitization in `buildSafeEnv()`
4. **Shell injection detection** in `containsShellInjection()`
5. **UID/GID dropping** to `nobody` on Linux when running as root

---

## 10. Summary

**Positives:**
- No circular dependencies detected
- Strong plugin isolation architecture
- Good use of AsyncGenerator for streaming
- Clear layer separation

**Concerns:**
- Missing error boundaries in critical paths (REPL, loop, tool execution)
- God objects in key files (PluginHost, runLoop, DaemonController)
- Engine layer should not depend on UI layer
- Several files exceed 400 lines

---

*Audit completed successfully. Total files analyzed: 50 TypeScript files.*
