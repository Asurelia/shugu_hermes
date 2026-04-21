# Error Handling Audit Report

**Date:** 2026-04-21  
**Scope:** `/tmp/shugu-loop/src/` — TypeScript files  
**Severity Guidelines:** Per AGENTS.md — treat silent failures as high severity unless clearly non-critical.

---

## VERIFIED ISSUES

### 1. Empty Catch Block — `voice/capture.ts` (Line 98)

**File:** `src/voice/capture.ts`  
**Function:** Anonymous cleanup function in promise executor  
**Severity:** MEDIUM  

**Code:**
```typescript
cleanup: async () => { try { await unlink(tempPath); } catch {} },
```

**Issue:** The `catch {}` block silently swallows any errors from the `unlink(tempPath)` call. While this cleanup is non-critical (temp file removal), silently ignoring errors could mask permission issues or other file system problems that might be relevant in debugging scenarios.

**Suggested Fix:** Add logging or at minimum acknowledge the error:
```typescript
cleanup: async () => {
  try { await unlink(tempPath); }
  catch (err) { /* ignore cleanup failures — temp file */ }
},
```

---

### 2. Silent Vault Write Failures — `knowledge-hook.ts` (Lines 53–55)

**File:** `src/plugins/builtin/knowledge-hook.ts`  
**Function:** `detectAndSaveMemoryHints()`  
**Severity:** LOW (per comment: vault writes are best-effort)  

**Code:**
```typescript
}).catch(() => {
  // Silent failure — vault writes are best-effort
});
```

**Issue:** Memory hints detected during conversation are silently discarded if vault writes fail. The comment explicitly acknowledges this is intentional, but losing auto-extracted memories without any warning could lead to unexpected behavior.

**Suggested Fix:** Consider logging at debug level, or emitting an event:
```typescript
}).catch((err) => {
  logger.debug('memory hint save failed (best-effort)', err);
});
```

---

### 3. Swallowed Git Errors in `/finish-feature` — `finish-feature.ts` (Lines 28, 33, 44)

**File:** `src/commands/finish-feature.ts`  
**Function:** `execute()`  
**Severity:** HIGH  

**Code:**
```typescript
const branch = (await git(['rev-parse', '--abbrev-ref', 'HEAD'], ctx.cwd).catch(() => 'HEAD')).trim();
const status = (await git(['status', '--porcelain'], ctx.cwd).catch(() => '')).trim();
const commitsRaw = await git(['log', `${mergeBase}..HEAD`, '--pretty=%H'], ctx.cwd).catch(() => '');
```

**Issue:** Git command failures silently return fallback values (`'HEAD'`, `''`). In `/finish-feature`, the command validates that the working tree is clean and that commits exist before merging. If git fails silently:
- A dirty working tree could be masked
- Missing commits could be hidden
- The wrong branch could be assumed

This is a **security-relevant** command (it merges to main) — silent failures here could cause unexpected behavior.

**Suggested Fix:** Fail explicitly with meaningful errors rather than using fallbacks:
```typescript
const branch = (await git(['rev-parse', '--abbrev-ref', 'HEAD'], ctx.cwd)
  .catch(() => { throw new Error('Failed to get current branch'); }))
```

---

### 4. Swallowed Git Errors in `/socratic` — `socratic.ts` (Lines 183–187)

**File:** `src/commands/socratic.ts`  
**Function:** Scope resolution in `execute()`  
**Severity:** MEDIUM  

**Code:**
```typescript
const diff = await git(['diff', `${base}..HEAD`], cwd).catch(() => '');
const commitsRaw = await git(['log', `${base}..HEAD`, '--pretty=%H'], cwd).catch(() => '');
const branch = (await git(['rev-parse', '--abbrev-ref', 'HEAD'], cwd).catch(() => 'HEAD')).trim();
const touched = await git(['diff', `${base}..HEAD`, '--name-only'], cwd).catch(() => '');
```

**Issue:** If git commands fail during diff scope resolution, the command returns an empty preloaded context. The user receives no indication that the diff was unavailable — it just looks like there's no diff.

**Suggested Fix:** If diff scope is requested but git fails, report an error:
```typescript
if (scope === 'diff') {
  let base = 'HEAD~1';
  try {
    const mergeBase = (await git(['merge-base', 'HEAD', 'main'], cwd)).trim();
    if (mergeBase) base = mergeBase;
  } catch {
    // main missing — keep HEAD~1 as fallback
  }
  const diff = await git(['diff', `${base}..HEAD`], cwd)
    .catch(() => { throw new Error('Failed to get diff — is this a valid git repository?'); });
  // ...
}
```

---

### 5. Swallowed Git Error in `/review` — `review.ts` (Line 25)

**File:** `src/commands/review.ts`  
**Function:** `execute()` — diff retrieval  
**Severity:** MEDIUM  

**Code:**
```typescript
diffContext = await git(['diff', 'HEAD~1'], ctx.cwd).catch(() => '');
```

**Issue:** When no staged/unstaged changes exist and diff against HEAD~1 also fails, the command silently continues and then returns an error "No code changes found". This masks the git failure and could confuse users.

**Suggested Fix:** Return explicit git errors:
```typescript
} catch (err) {
  const msg = err instanceof Error ? err.message : String(err);
  return { type: 'error', message: `git error: ${msg}` };
}
```

---

### 6. Silent Session Load Failure in Bootstrap — `bootstrap.ts` (Line 552)

**File:** `src/entrypoints/bootstrap.ts`  
**Function:** Session rehydration in main bootstrap  
**Severity:** MEDIUM  

**Code:**
```typescript
? await sessionMgr.load(cliArgs.resumeSession).catch(() => null)
```

**Issue:** If a user specifies `--resume-session <id>` but loading fails, the session silently falls back to a fresh session with no warning to the user. This could cause data loss if the user intended to resume.

**Suggested Fix:** Log at minimum a warning:
```typescript
const resumed = await sessionMgr.load(cliArgs.resumeSession)
  .catch((err) => {
    logger.warn(`failed to resume session ${cliArgs.resumeSession}`, err);
    return null;
  });
```

---

### 7. Silent Intelligence Feature Failures — `intelligence.ts` (Lines 324, 327, 334)

**File:** `src/engine/intelligence.ts`  
**Function:** `runIntelligence()`  
**Severity:** MEDIUM  

**Code:**
```typescript
? generatePromptSuggestion(client, messages, intelligenceModel).catch(() => null)
// ...
? extractMemories(client, messages, intelligenceModel).catch(() => [])
// ...
speculation = await speculate(client, suggestion, messages, intelligenceModel).catch(() => null);
```

**Issue:** If intelligence features (suggestions, memory extraction, speculation) fail, they silently return null/empty array. The system continues without these features. While this is somewhat intentional (these are "nice to have" features), users may be confused when suggestions or memories don't appear with no indication why.

**Suggested Fix:** Log failures at debug level so operators can investigate:
```typescript
? generatePromptSuggestion(client, messages, intelligenceModel)
    .catch((err) => { logger.debug('suggestion failed', err); return null; })
```

---

### 8. Silent Plugin Loading in Meta Mode — `runtime.ts` (Line 89)

**File:** `src/meta/runtime.ts`  
**Function:** `runMetaMode()`  
**Severity:** LOW  

**Code:**
```typescript
await pluginRegistry.loadAll(cwd, registry, /* no commands */ undefined as never, /* no skills */ undefined as never, {
  onConfirmLocal: async () => true, // auto-accept in meta mode
}).catch(() => { /* plugin loading is non-critical */ });
```

**Issue:** Comment states plugin loading is non-critical for meta mode, but silently swallowing failures could mask real issues that affect harness behavior. If plugins fail to load, the harness may behave differently than expected without operators knowing.

**Suggested Fix:** Log at info level:
```typescript
.catch((err) => { logger.info('plugin loading skipped in meta mode (non-critical)'); });
```

---

### 9. Silent Shutdown Fallback — `registry.ts` (Lines 171, 187)

**File:** `src/plugins/registry.ts`  
**Function:** `unload()` and `disposeAll()`  
**Severity:** LOW (fallback behavior is reasonable)  

**Code:**
```typescript
host.shutdown().catch(() => host.kill());
// and
await host.shutdown().catch(() => host.kill());
```

**Issue:** If graceful shutdown fails, it falls back to kill. This is actually reasonable fallback behavior, though the kill could be abrupt. The catch is intentionally falling back rather than truly swallowing.

**Assessment:** Acceptable — the fallback to `kill()` ensures cleanup happens. The error is effectively handled.

---

### 10. Silent IPC Disconnect — `daemon.ts` (Lines 207–209)

**File:** `src/automation/daemon.ts`  
**Function:** `stop()`  
**Severity:** LOW  

**Code:**
```typescript
try {
  this.child.send({ type: 'stop', timestamp: new Date().toISOString(), nonce: this.ipcNonce ?? undefined });
} catch {
  // IPC might be disconnected
}
```

**Issue:** If IPC is already disconnected when stop is called, the error is silently ignored. This is generally acceptable since the daemon is stopping anyway, but some logging would aid debugging.

**Suggested Fix:** Log at debug level:
```typescript
} catch (err) {
  logger.debug('IPC send failed during stop (may already be disconnected)', err);
}
```

---

### 11. Silent Interrupted Error Swallow — `proactive.ts` (Lines 179–182)

**File:** `src/automation/proactive.ts`  
**Function:** `run()` generator  
**Severity:** LOW  

**Code:**
```typescript
} catch (error) {
  if (!this.interrupt.aborted) {
    throw error;
  }
}
```

**Issue:** If an error occurs but the interrupt was also triggered, the error is silently swallowed. This is intentional but could mask issues during controlled shutdown.

**Assessment:** Context-dependent — during normal interruption this is acceptable. Consider logging in debug mode.

---

## SUMMARY

| Severity | Count | Issue |
|----------|-------|-------|
| **HIGH** | 1 | Swallowed git errors in `/finish-feature` |
| **MEDIUM** | 5 | `voice/capture.ts` empty catch, `/socratic` git errors, `/review` git error masking, bootstrap session silent fail, intelligence silent fail |
| **LOW** | 5 | knowledge-hook silent vault fail, meta runtime silent plugin fail, registry shutdown fallback, daemon IPC silent fail, proactive interrupted swallow |

**Total Verified Issues:** 11

**By Category:**
- Empty catch blocks: 1
- Silent failures with `.catch(() => ...)`: 8
- Fallback-with-no-warning patterns: 2

---

## RECOMMENDED ACTIONS

1. **High Priority:** Fix git error swallowing in `/finish-feature` — this is a security-relevant command that merges to main
2. **Medium Priority:** Add logging/debug output to silent failures in intelligence features and bootstrap session loading
3. **Low Priority:** Add debug logging to acceptable silent failures (IPC, shutdown) for operator visibility

---

## BLIND SPOTS & UNVERIFIED

- Could not verify runtime behavior of plugin brokered isolation error propagation
- Hook error escalation in `plugins/host.ts` appears correct (proper JSON-RPC error responses) but was not runtime-tested
- Session manager error paths in `context/session/` directory were not audited in detail
- Transport layer (`transport/`) error handling not audited in this pass
