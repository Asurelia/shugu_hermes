# Shugu Performance Audit

**Date:** 2026-04-19
**Scope:** `/tmp/shugu-loop/src/`
**Categories:** N+1 Query Patterns, Memory Leaks (EventEmitter), Inefficient Loops/Algorithms

---

## Summary

| Severity | Category | Count |
|----------|----------|-------|
| High | N+1 Query Patterns | 7 |
| Medium | Memory Leak (EventEmitter listeners not removed) | 4 |
| Low | Inefficient Algorithm | 2 |

---

## HIGH — N+1 Query Patterns

Multiple methods in `ObsidianVault` and `MemoryStore` read files sequentially inside loops instead of batch/parallel fetching.

---

### 1. `ObsidianVault.searchContent()` — Sequential note reads per query

**File:** `src/context/memory/obsidian.ts`, lines 120–153

```typescript
async searchContent(query: string, limit: number = 10): Promise<ObsidianNote[]> {
  const allNotes = await this.listNotes();          // 1 call
  for (const notePath of allNotes) {
    const note = await this.readNote(notePath);    // N sequential calls!
    ...
  }
}
```

**Impact:** Every search iterates all vault notes and calls `readFile` sequentially. For a vault with 500 notes, that is 501 filesystem calls per query.

**Suggested fix:** Replace the sequential `for` loop with `Promise.all` over the note paths, or batch-read notes in groups. Since `readNote` is I/O-bound, parallelizing provides near-linear speedup:

```typescript
const notes = await Promise.all(allNotes.map(p => this.readNote(p)));
const scored = notes.filter(Boolean).map(note => { ... });
```

---

### 2. `ObsidianVault.getStaleNotes()` — Sequential stat + read per note

**File:** `src/context/memory/obsidian.ts`, lines 368–390

```typescript
for (const notePath of allNotes) {
  const s = await stat(absPath);              // N sequential calls
  if (s.mtimeMs < cutoff) {
    const note = await this.readNote(notePath); // more sequential I/O
  }
}
```

**Impact:** One `stat` per note path, then potentially one `readFile`. With 1000 notes this is 1000+ sequential syscalls.

**Suggested fix:** Use `Promise.all` for both the `stat` calls and the `readNote` calls.

---

### 3. `ObsidianVault.getRecentNotes()` — Same pattern

**File:** `src/context/memory/obsidian.ts`, lines 454–476

Same N+1 issue: sequential `stat` then `readNote` per note.

---

### 4. `ObsidianVault.getLinkedNotes()` — Nested sequential reads

**File:** `src/context/memory/obsidian.ts`, lines 192–206

```typescript
async getLinkedNotes(notePath: string): Promise<ObsidianNote[]> {
  const note = await this.readNote(notePath);      // 1 read
  for (const link of note.links) {
    const resolved = await this.resolveLink(link);
    const linkedNote = await this.readNote(resolved); // N sequential reads
  }
}
```

`resolveLink` itself (lines 177–187) is also N+1: it calls `listNotes()` then loops with `readNote`.

---

### 5. `ObsidianVault.resolveLink()` — Sequential read per path

**File:** `src/context/memory/obsidian.ts`, lines 177–187

```typescript
const allNotes = await this.listNotes();
for (const notePath of allNotes) {
  const note = await this.readNote(notePath);  // N sequential reads to find one
}
```

**Impact:** Every wikilink resolution does a full vault scan with sequential reads. A note with 10 wikilinks = 10 × (1 `listNotes` + N `readNote`) = very expensive.

**Suggested fix:** Build a title→path index once (on `listNotes` cache population) and do O(1) lookups.

---

### 6. `ObsidianVault.searchByTag()` — Sequential read per note

**File:** `src/context/memory/obsidian.ts`, lines 158–172

```typescript
for (const notePath of allNotes) {
  const note = await this.readNote(notePath);  // N sequential reads
  if (note.tags.some(t => t === normalizedTag)) { ... }
}
```

---

### 7. `MemoryAgent.migrateOldMemories()` — Sequential file reads in a loop

**File:** `src/context/memory/agent.ts`, lines 465–501

```typescript
for (const file of files) {
  if (!file.endsWith('.md')) continue;
  const content = await readFile(join(oldDir, file), 'utf-8'); // sequential
  ...
}
```

**Suggested fix:** Use `Promise.all(files.map(file => readFile(...)))` for parallel reads.

---

### 8. `MemoryStore.updateIndex()` — Reloads all files just to write an index

**File:** `src/context/memory/store.ts`, lines 165–175

```typescript
async save(memory: Omit<Memory, 'filename'>): Promise<string> {
  await writeFile(tmpPath, content, ...);
  await this.updateIndex(dir);  // reads ALL files again
}

private async updateIndex(dir: string): Promise<void> {
  const memories = await this.loadFromDir(dir); // reads ALL .md files
  // ...writes index
}
```

**Impact:** Every single memory save triggers a full scan of the memory directory. With 100 memories, each save reads 100 files.

**Suggested fix:** Maintain the index incrementally (add one line on save, rebuild only on load).

---

## MEDIUM — Memory Leaks (EventEmitter listeners not detached)

---

### 9. `DaemonController.stop()` — 'exit' listener added without explicit remove

**File:** `src/automation/daemon.ts`, lines 211–226

```typescript
await new Promise<void>((resolve) => {
  const timeout = setTimeout(() => {
    this.child?.kill('SIGKILL');
    resolve();
  }, 5000);

  this.child!.on('exit', () => {  // ← listener added
    clearTimeout(timeout);
    resolve();
  });

  this.child!.kill('SIGTERM');
});

this.detachChildListeners();  // bulk removal via removeAllListeners
```

**Impact:** The `stop()` method adds an 'exit' listener inside a Promise. While `detachChildListeners()` calls `removeAllListeners('exit')` afterward, this is a bulk removal — not an explicit `removeListener` for this specific handler. If multiple `stop()` calls race (unlikely but possible), listener state can become inconsistent. More importantly, the explicit pattern of adding a listener and relying on bulk removal is fragile and not self-documenting.

**Suggested fix:** Store a reference to the handler and call `this.child!.removeListener('exit', handler)` explicitly after the Promise settles.

---

### 10. `PluginRegistry` — 'crashed' listener on host never removed on unload

**File:** `src/plugins/registry.ts`, lines 93–100

```typescript
const host = (plugin as PluginWithHost)._host;
if (host) {
  host.on('crashed', () => {
    this.unload(plugin.manifest.name);
    this.emit('plugin:crashed', plugin.manifest.name);
  });
}
```

**Impact:** When a plugin is unloaded via `unload()`, the host's 'crashed' listener is **never removed**. The host object may still be referenced and its listeners prevent garbage collection. On repeated load/unload cycles for the same plugin, multiple 'crashed' handlers accumulate.

**Suggested fix:** In `unload()`, call `host.removeListener('crashed', handler)` or use a named method reference that can be explicitly unbound.

---

### 11. `PluginRegistry` — In `disposeAll()`, hosts are shut down but listeners persist

**File:** `src/plugins/registry.ts`, lines 183–190

```typescript
async disposeAll(): Promise<void> {
  for (const plugin of this.plugins.values()) {
    const host = (plugin as PluginWithHost)._host;
    if (host && !host.isDead) {
      await host.shutdown().catch(() => host.kill());
    }
  }
}
```

**Impact:** `disposeAll()` shuts down hosts but does not remove the 'crashed' listeners registered in `loadAll()`. If the registry is re-used after `disposeAll()` (e.g., in tests), stale listeners remain.

**Suggested fix:** Track listener references and remove them before shutdown, or switch to a weak map pattern.

---

### 12. `DaemonWorker` — Named handler but `process.off()` uses correct reference

**File:** `src/automation/daemon.ts`, lines 390–416, 451–456

```typescript
this.messageHandler = (msg: DaemonMessage) => { ... };
process.on('message', this.messageHandler);

stop(): void {
  if (this.messageHandler) {
    process.off('message', this.messageHandler);  // ← correct: named reference
    this.messageHandler = null;
  }
}
```

**Verdict:** This one is correct — the handler is stored as `this.messageHandler` and explicitly removed. No issue here.

---

## LOW — Inefficient Algorithms

---

### 13. `expandQueryTerms()` — O(q × m × k) nested loops with `some()` calls

**File:** `src/context/memory/agent.ts`, lines 75–90

```typescript
for (const word of queryWords) {           // q iterations
  for (const group of TERM_GROUPS) {        // m = 15 groups
    if (group.some(term => term.includes(word) || word.includes(term))) {
      for (const term of group) {           // k ≈ 10 terms/group
        expanded.add(term);
      }
    }
  }
}
```

**Impact:** For a query with 5 words and 15 groups of ~10 terms each, this does ~750 `String.includes()` calls. `TERM_GROUPS` is static and could be pre-indexed once.

**Suggested fix:** Pre-build a `wordToGroups: Map<string, Set<number>>` at module load time, mapping each term to its group index. Then lookup is O(1) per word instead of O(m × k).

---

### 14. `DaemonController.startHeartbeat()` — Heartbeat timer not cleared in all paths

**File:** `src/automation/daemon.ts`, lines 329–343

```typescript
private startHeartbeat(): void {
  this.heartbeatTimer = setInterval(() => {
    if (this.child && !this.child.killed) { this.child.send({ type: 'heartbeat', ... }); }
    else { /* swallow */ }
  }, 30_000);
}
```

**Impact:** `stopHeartbeat()` (lines 345–350) properly clears the interval, but if the daemon crashes (not via `stop()`), the interval timer is not cleared. The timer is `.unref()`-ed so it doesn't prevent exit, but it does consume resources.

**Suggested fix:** Ensure `stopHeartbeat()` is called in both the normal shutdown path and in the 'exit' handler.

---

## Verified Clean (No Issues Found)

The following files/modules were reviewed and have no significant performance issues:

- **`plugins/host.ts`** — Properly cleans up all child listeners via `detachChildListeners()` called from the 'exit' handler. Well-documented.
- **`plugins/hooks.ts`** — `HookRegistry` emits events but listeners are managed by the caller (PluginRegistry). No direct leak pattern.
- **`utils/tracer.ts`** — `onEvent()` returns an explicit unsubscribe function that calls `_emitter.off()`. Correct pattern.
- **`credentials/prompt.ts`** — `readHidden()` properly removes the 'data' listener via `cleanup()`.
- **`ui/paste.ts`** — `disable()` properly restores `origEmit` and removes the stdin listener.
- **`automation/proactive.ts`** — Uses `AbortController` pattern correctly. No lingering listeners.
- **`context/session/work-context.ts`** — Pure function, no side effects, no I/O.
- **`context/compactor.ts`** — Functional approach, no N+1 issues.
- **`agents/delegation.ts`** — Parallel delegation uses `Promise.all` correctly.

---

## Recommendations (Priority Order)

1. **High:** Parallelize `ObsidianVault.searchContent()`, `getStaleNotes()`, `getRecentNotes()`, `getLinkedNotes()`, and `resolveLink()`. These are called frequently and each does sequential I/O.
2. **High:** Fix `MemoryStore.updateIndex()` to maintain the index incrementally instead of rebuilding from all files on every save.
3. **High:** In `PluginRegistry.unload()`, remove the 'crashed' listener from the host before shutting down.
4. **Medium:** Pre-index `TERM_GROUPS` for `expandQueryTerms()` to reduce per-search CPU cost.
5. **Medium:** Add explicit `removeListener` call in `DaemonController.stop()` instead of relying on bulk `removeAllListeners`.
6. **Low:** Ensure `stopHeartbeat()` is called in the 'exit' path of `DaemonController`.
