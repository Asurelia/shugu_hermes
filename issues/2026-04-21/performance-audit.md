# Performance Audit — Shugu-loop

**Date:** 2026-04-21  
**Scope:** `/tmp/shugu-loop/src/` — TypeScript files  
**Auditor:** Hermes Agent

---

## Summary

| Category | Severity | Count |
|----------|----------|-------|
| Memory Leaks (EventEmitter) | Medium | 2 |
| N+1 Query Patterns | Low | 2 |
| Inefficient Loops/Algorithms | Low | 4 |
| Process Listener Leaks | Low | 2 |

---

## 1. Memory Leaks — EventEmitter Listeners Not Detached

### 1.1 `automation/daemon.ts` — process.on('message') not removed on restart

**File:** `src/automation/daemon.ts:416`  
**Severity:** Medium  
**Impact:** If the daemon restarts (auto-restart on crash), the previous `message` listener accumulates. Each restart adds a new listener without removing the old one.

```typescript
// Line 416
process.on('message', this.messageHandler);
```

**Detected at:** `src/automation/daemon.ts:453-455` — listener is removed in `stop()`, but if `start()` is called again after `stop()`, a new listener is added without removing the old one.

**Suggested Fix:** Track listener removal state and ensure `process.off()` is called before re-registering.

---

### 1.2 `entrypoints/repl.ts` — Signal handlers not removed

**File:** `src/entrypoints/repl.ts:318-327`  
**Severity:** Low  
**Impact:** `process.on('SIGINT')`, `process.on('SIGTERM')`, `process.on('SIGHUP')` are registered but never removed during REPL lifetime. Since REPL is a long-lived process that doesn't restart internally, this is less critical.

```typescript
process.on('SIGINT', () => { ... });
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGHUP', () => gracefulShutdown('SIGHUP'));
```

**Note:** These are intentionally kept for process lifecycle. Low severity since the process exits anyway.

---

### 1.3 GOOD: `plugins/host.ts` — Properly detaches child listeners

**File:** `src/plugins/host.ts:416-428`  
**Positive Finding:** The `detachChildListeners()` method properly cleans up all EventEmitter listeners on child process exit. Comment at lines 408-415 explicitly documents the memory leak risk this prevents.

---

### 1.4 GOOD: `utils/tracer.ts` — Returns unsubscribe function

**File:** `src/utils/tracer.ts:489-492`  
**Positive Finding:** `onEvent()` returns a cleanup function for proper listener removal.

---

## 2. N+1 Query Patterns

### 2.1 `context/memory/store.ts` — findRelevant() loads all memories

**File:** `src/context/memory/store.ts:100-119`  
**Severity:** Low  
**Impact:** `findRelevant()` calls `loadAll()` which loads ALL memory files (global + project) before filtering by relevance. For large memory stores, this is inefficient.

```typescript
async findRelevant(query: string, limit: number = 5): Promise<Memory[]> {
  const all = await this.loadAll(); // Loads ALL files
  const scored = all.map((memory) => { ... }); // Then scores ALL
  return scored.filter(...).sort(...).slice(0, limit);
}
```

**Suggested Fix:** Consider indexing memories at startup, or implementing a scoring threshold to short-circuit early.

---

### 2.2 `context/memory/store.ts` — loadFromDir() sequential file reads

**File:** `src/context/memory/store.ts:124-144`  
**Severity:** Low  
**Impact:** `loadFromDir()` reads each `.md` file sequentially in a `for...of` loop with `await` per file. For directories with many memories, this is slower than `Promise.all()`.

```typescript
for (const file of files) {
  if (!file.endsWith('.md') || file === 'MEMORY.md') continue;
  try {
    const content = await readFile(join(dir, file), 'utf-8'); // Sequential!
    const memory = this.parseMemoryFile(content, file);
    if (memory) memories.push(memory);
  } catch { /* skip */ }
}
```

**Suggested Fix:** Filter files first, then use `Promise.all()` to read in parallel.

---

## 3. Inefficient Loops/Algorithms

### 3.1 `context/session/work-context.ts` — Full message history scan per extraction

**File:** `src/context/session/work-context.ts:66`  
**Severity:** Low  
**Impact:** `extractWorkContext()` iterates backwards through the ENTIRE message array on every call. For long sessions, this is O(n) per extraction.

```typescript
for (let i = messages.length - 1; i >= 0 && toolHistory.length < maxHistory; i--) {
  const msg = messages[i]!;
  // ... scans all messages
}
```

**Suggested Fix:** Track `lastProcessedIdx` and resume from there on subsequent calls.

---

### 3.2 `context/memory/store.ts` — O(n log n) sort instead of heap for top-k

**File:** `src/context/memory/store.ts:115-118`  
**Severity:** Low  
**Impact:** `findRelevant()` sorts all scored memories before slicing to `limit`. This is O(n log n) when only the top-k elements are needed.

```typescript
return scored
  .filter((s) => s.score > 0)
  .sort((a, b) => b.score - a.score)  // Full sort!
  .slice(0, limit);
```

**Suggested Fix:** Use a min-heap of size `limit` for O(n log k) complexity.

---

### 3.3 `plugins/loader.ts` — Sequential stat calls per directory

**File:** `src/plugins/loader.ts:60-67` (in `detectProject`)  
**Severity:** Low  
**Impact:** `detectProject()` calls `stat()` sequentially for each directory in a loop.

```typescript
for (const d of dirs) {
  if (entries.includes(d)) {
    try {
      const s = await stat(join(cwd, d)); // Sequential stat calls
      if (s.isDirectory()) info.structure.push(d);
    } catch { /* skip */ }
  }
}
```

**Suggested Fix:** Use `Promise.all()` for parallel stat calls.

---

### 3.4 `transport/stream.ts` — while(true) with string concatenation

**File:** `src/transport/stream.ts:39-47`  
**Severity:** Low  
**Impact:** Buffer uses string concatenation (`buffer += decoder.decode(...)`) which creates new string objects repeatedly. For large streams, this causes GC pressure.

```typescript
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  buffer += decoder.decode(value, { stream: true }); // String concat in loop
```

**Suggested Fix:** Use an array buffer approach or `TextDecoderStream` for better performance.

---

## 4. Observations

### Good Patterns Found

1. **PluginHost listener cleanup** (`plugins/host.ts:416-428`): Well-documented listener detachment with explicit comment explaining the memory leak risk.

2. **Tracer unsubscribe pattern** (`utils/tracer.ts:489-492`): Returns cleanup function from `onEvent()`.

3. **DaemonController messageHandler cleanup** (`automation/daemon.ts:453-455`): `process.off('message', this.messageHandler)` is properly called.

---

## 5. Recommendations

| Priority | Issue | File | Suggested Fix |
|----------|-------|------|---------------|
| High | Listener leak on daemon restart | `daemon.ts:416` | Track listener state; remove before re-registering |
| Medium | `loadFromDir()` sequential reads | `store.ts:129-138` | Use `Promise.all()` after filtering |
| Medium | `findRelevant()` loads all | `store.ts:101` | Add indexing or early termination |
| Low | String concat in SSE buffer | `stream.ts:47` | Use array buffer approach |
| Low | Full sort for top-k | `store.ts:115-118` | Use min-heap of size `limit` |

---

*Audit completed successfully. Total files analyzed: 50 TypeScript files.*
