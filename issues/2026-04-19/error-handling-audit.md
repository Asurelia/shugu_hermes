# Shugu Error Handling Audit

**Date:** 2026-04-19
**Scope:** `/tmp/shugu-loop/src/` — TypeScript source
**Patterns searched:** `catch {}`, `catch { ... }`, `.catch()`, `try/catch`

---

## Summary

| Severity | Count | Description |
|---|---|---|
| **HIGH** | 4 | Empty catch blocks swallowing plugin/manifest load failures, Docker check, and vault writes |
| **MEDIUM** | 6 | Git commands returning fallbacks on failure, masking real errors in critical commands |
| **LOW** | 4 | Well-commented, intentional silent failures in non-critical paths |
| **GOOD** | 5 | Errors properly propagated, logged, or escalated |

Total `catch` sites examined: ~60 across ~40 files.

---

## HIGH — Empty Catch / Swallowed Errors

### 1. `plugins/loader.ts:161` — Plugin manifest load swallows all errors

```typescript
} catch {
  return null;
}
```

**Function:** `loadManifest()`
**Impact:** If a manifest is malformed JSON, or unreadable, the plugin silently returns `null` and is skipped. No logging, no warning, no escalation.
**Suggested fix:** Log at `warn` level before returning `null`, including the error message and manifest path. Consider surfacing to the user via the plugin registry status.

---

### 2. `plugins/host.ts:105` — Docker availability check silently swallows errors

```typescript
} catch {
  _dockerAvailable = false;
}
```

**Function:** `isDockerAvailable()`
**Impact:** If `docker version` fails for any reason other than "not installed" (e.g., permission denied, daemon unreachable), the error is hidden and `_dockerAvailable` is incorrectly set to `false`. Plugin features that depend on Docker will silently behave incorrectly.
**Suggested fix:** Log the actual error at `debug` or `info` level. Distinguish ENOENT (not installed) from other failures.

---

### 3. `plugins/builtin/knowledge-hook.ts:53` — Vault writes fail silently

```typescript
}).catch(() => {
  // Silent failure — vault writes are best-effort
});
```

**File:** `knowledge-hook.ts`, `registerKnowledgeHooks()`
**Impact:** Fire-and-forget memory hint saves to Obsidian vault. If the vault write fails, the error is silently discarded. No logging, no retry, no user notification. While "best-effort" memory is a reasonable philosophy, zero visibility into repeated failures is not.
**Suggested fix:** At minimum log at `debug` level. Consider a counter or periodic warning if failure rate is high.

---

### 4. `plugins/child-entry.ts:411` — stdin JSON parse failure is stderr-only

```typescript
} catch {
  process.stderr.write(`[plugin-child] Invalid JSON on stdin\n`);
}
```

**Function:** `rl.on('line', ...)` — plugin child process stdin handler
**Impact:** JSON parse errors on the plugin-to-parent communication channel write only to stderr and then silently continue. Malformed messages are dropped with no recovery, no error escalation to the parent plugin host.
**Suggested fix:** Emit an error event back to the parent or write a structured error line so the host can observe and log it properly.

---

## MEDIUM — Git Command Failures Masked with Empty Fallbacks

### 5. `commands/finish-feature.ts:28` — `git rev-parse` failure → `'HEAD'`

```typescript
const branch = (await git(['rev-parse', '--abbrev-ref', 'HEAD'], ctx.cwd).catch(() => 'HEAD')).trim();
```

**Impact:** If `git rev-parse` fails (e.g., not a git repo, detached HEAD), the command silently substitutes `'HEAD'`. The subsequent `PROTECTED_BRANCHES` check then runs against `'HEAD'` instead of the real branch name — potentially bypassing the protected-branch guard on main/master.
**Severity:** A `git rev-parse` failure is rare but meaningful. Treating it as `'HEAD'` can lead to merging from an unintended branch.

---

### 6. `commands/finish-feature.ts:33` — `git status` failure → `''`

```typescript
const status = (await git(['status', '--porcelain'], ctx.cwd).catch(() => '')).trim();
```

**Impact:** A failing `git status` is treated as an empty (clean) working tree. If `git` cannot run due to errors, the command proceeds as if the tree is clean — potentially proceeding to merge with uncommitted changes.

---

### 7. `commands/finish-feature.ts:44` — `git log` failure → `''`

```typescript
const commitsRaw = await git(['log', `${mergeBase}..HEAD`, '--pretty=%H'], ctx.cwd).catch(() => '');
```

**Impact:** If the commit log query fails, `commits` becomes an empty array. The command then reports "no commits since main — nothing to finish" even if the failure was due to a git error rather than an empty history.

---

### 8. `commands/socratic.ts:183–188` — Multiple git fallbacks

```typescript
const diff = await git(['diff', `${base}..HEAD`], cwd).catch(() => '');
const commitsRaw = await git(['log', `${base}..HEAD`, '--pretty=%H'], cwd).catch(() => '');
const branch = (await git(['rev-parse', '--abbrev-ref', 'HEAD'], cwd).catch(() => 'HEAD')).trim();
const touched = await git(['diff', `${base}..HEAD`, '--name-only'], cwd).catch(() => '');
```

**Impact:** Same pattern as `finish-feature`. The `/socratic` command produces preloaded context for a diff review. If git fails, the preloaded context will be empty or partial. The model then proceeds with incomplete context but no error signal to the user.
**Suggested fix for all git fallbacks:** Catch specific git errors, log them, and return an error result rather than silently substituting an empty string that looks like valid output.

---

### 9. `commands/socratic.ts:180` — `merge-base` failure fallback to `HEAD~1`

```typescript
} catch {
  // main missing — keep HEAD~1 as fallback
}
```

**Impact:** If `main` branch doesn't exist, the command silently uses `HEAD~1` as the base. For a command designed to review diffs, using the wrong base could miss significant history. The comment says "main missing" but this could also be any other git error.

---

## LOW — Intentional Silent Failures (Non-Critical Paths)

### 10. `plugins/host.ts:390` — Plugin shutdown: graceful fail → force kill

```typescript
} catch {
  // If shutdown request fails, force kill
  this.kill();
}
```

**Verdict:** Acceptable. The shutdown request is advisory; if it fails, force-killing the child process is the correct fallback. The error is not silently absorbed — it's acted upon.

---

### 11. `plugins/broker.ts:94–104` — Path resolution fallback

```typescript
} catch {
  // File may not exist yet (for writes).
  // Resolve the PARENT directory to prevent symlink/traversal attacks,
  // then join the leaf filename back onto the validated parent.
```

**Verdict:** Correct behavior. The file may not exist for writes, and the fallback correctly validates the parent directory to prevent traversal attacks before accepting the path. The second `catch` at line 103 properly escalates with an error.

---

### 12. `skills/bundled/vibe.ts:217` — Workflow JSON load failure → `null`

```typescript
} catch {
  return null;
}
```

**Verdict:** Low severity. If a workflow file is corrupted, returning `null` causes the workflow to be treated as absent. This is a reasonable fallback for a non-critical feature. Could log at `debug` level.

---

### 13. `skills/bundled/vibe.ts:353` — Unreadable output files skipped

```typescript
} catch {
  // Skip unreadable outputs
}
```

**Verdict:** Acceptable for build artifacts. If an output file is unreadable, skipping it and continuing is reasonable. The overall workflow context is still built from readable files.

---

### 14. `commands/trace.ts:154` — Missing subdirectory skipped

```typescript
} catch {
  // missing — skip
}
```

**Verdict:** Low severity. Tracing agent/trace directories that disappear between listing and stat is a race condition that's safe to skip.

---

## GOOD — Errors Properly Handled

### 15. `plugins/policy.ts:110–123` — Policy load with proper error differentiation

```typescript
} catch (err) {
  if ((err as NodeJS.ErrnoException).code === 'ENOENT') {
    return null;  // Policy absent — not an error
  }
  logger.warn(`plugin-policy: failed to read ${policyPath}`, ...);
  return null;
}
```

**Verdict:** Good. ENOENT is treated as "no policy" (normal). Other errors are logged with context.

---

### 16. `transport/stream.ts:68–71` — SSE parse failures logged at debug

```typescript
} catch (err) {
  logger.debug('SSE JSON parse failed', `${jsonStr.slice(0, 80)} | ...`);
}
```

**Verdict:** Good. Partial SSE lines from MiniMax are expected. Debug-level logging is appropriate.

---

### 17. `transport/stream.ts:372–375` — AbortError swallowed with explicit comment

```typescript
if (err instanceof Error && err.name === 'AbortError') {
  return;
}
throw err;
```

**Verdict:** Good. The comment explicitly justifies swallowing AbortError — matching StreamAbortError and DOM AbortException.

---

### 18. `plugins/hooks.ts:137–139` — Hook errors logged and traced

```typescript
} catch (error) {
  logger.warn(`Hook PreToolUse from plugin "${hook.pluginName}" threw: ${(error as Error).message}`);
  tracer.log('error', { hook: hook.pluginName, type: 'PreToolUse', error: (error as Error).message });
}
```

**Verdict:** Good. Errors from third-party plugin hooks don't crash the tool chain but are surfaced via logger and tracer.

---

### 19. `plugins/host.ts:475–477` — Malformed JSON from plugin escalated

```typescript
} catch {
  this.emit('error', new Error(`Malformed JSON from plugin "${this.pluginName}": ${line.slice(0, 100)}`));
  return;
}
```

**Verdict:** Good. Malformed messages are turned into explicit error events so the host can observe and handle them.

---

## BLIND SPOTS & UNVERIFIED

- **Plugin child process message handling** (`plugins/child-entry.ts:408`): `handleMessage(msg).catch(...)` — the catch is visible but the full `handleMessage` path was not traced in detail.
- **Trigger server request handling** (`automation/triggers.ts:94`): `this.handleRequest(...).catch((err) => { this.sendJson(res, 500, { error: 'Internal server error' }); });` — the error object is discarded; the client gets a generic 500 with no error ID for correlation.
- **`skills/bundled/loop.ts:128`**: `runOnce().catch(...)` — first-iteration failures are logged via `ctx.error()` which is good, but if `ctx` is not available in some scenarios this could be a silent failure.
- **Companion init** (`entrypoints/cli-handlers.ts:24`): `try { _companion = getCompanion(); } catch { _companion = null; }` — companion failure silently falls back to `null`. This is likely acceptable for non-critical UI, but verified against actual usage paths.
- **Trust store** (`credentials/trust-store.ts:132–139`): Read failures are logged only if not `ENOENT`; write failures at line 147 have no error handling at all — if `writeStore` throws, the caller gets no feedback.

---

## RECOMMENDATIONS

1. **Add logging to all silent `catch {}` blocks** in plugin/manifest loading and vault writes. At minimum use `logger.debug()` so operators can enable visibility.

2. **Distinguish ENOENT from other errors** in `loadManifest()` and `isDockerAvailable()` — only ENOENT is truly silent; permission errors and corruption should be surfaced.

3. **Git command fallbacks in `finish-feature` and `socratic`** should return error results rather than empty strings that look like valid output. The calling code cannot distinguish "git failed" from "git returned empty."

4. **`child-entry.ts` stdin error handling** should emit a structured error to the parent host rather than only writing to the child's stderr.

5. **`writeStore` in `trust-store.ts`** (line 147) has no try/catch — if writing the trust store fails, the approval is lost silently. Add error handling.

6. **Trigger server** should include a correlation ID or log the actual error server-side when `handleRequest` rejects, rather than returning a generic 500.

---

*Audit conducted by Hermes agent. Patterns: `catch\s*\{`, `\.catch\(`, `catch\s*\(\s*\)` across all `.ts` files in `/tmp/shugu-loop/src/`.*
