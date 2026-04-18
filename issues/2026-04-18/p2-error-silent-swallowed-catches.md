# [P2] — Error Handling: Silent empty catch blocks swallow failures

**File:** Multiple files (see below)
**Line:** Various
**Severity:** P2 (Medium)
**Category:** Error Handling

## Description

Multiple locations use empty `catch {}` blocks that silently swallow errors without logging or fallback behavior. While many are in non-critical fallback paths (e.g., "skip unreadable files"), several occur in security-sensitive code paths where silent failure could hide attacks or misconfigurations.

## Evidence

### Critical silent catches:

1. **`src/tools/obsidian/ObsidianTool.ts:170`**
   ```typescript
   try { frontmatter = JSON.parse(fmStr); } catch { frontmatter = undefined; }
   ```
   JSON parse failure is silently ignored — malformed frontmatter is simply missing.

2. **`src/tools/bash/shells.ts:43`** — `whichBinary()`
   ```typescript
   } catch {
     return null;
   }
   ```
   Shell detection failure is completely silent — system misconfiguration hidden.

3. **`src/tools/agents/AgentTool.ts:150`**
   ```typescript
   } catch { detail = ''; } break; }
   ```
   URL parsing for WebFetch hostname extraction silently defaults to empty.

4. **`src/utils/network.ts:52` and `src/utils/network.ts:76`** — SSRF redirect handling
   ```typescript
   } catch {
     // Fallback to local
   }
   ```
   Network failures in redirect following silently fall back — SSRF detection could be bypassed without any warning.

5. **`src/utils/logger.ts:43,59`** — Logging failures
   ```typescript
   } catch { /* log silently */ }
   ```
   Logger failures are swallowed — if logging breaks, there's no indication.

6. **`src/transport/stream.ts:230`** — SSE stream parsing
   ```typescript
   } catch {
     buffer = '';
   }
   ```
   Stream parsing errors silently reset buffer — malformed server events hidden.

7. **`src/tools/web/WebSearchTool.ts:135`**, **`src/tools/web/WebFetchTool.ts:66`**
   Empty catches on API/network calls — errors hidden from user.

8. **`src/tools/files/FileWriteTool.ts:93`**
   ```typescript
   } catch {
     // Silently skip
   }
   ```
   Write failures silently skipped.

9. **`src/tools/search/GlobTool.ts:114,143`** — Native fallback
   ```typescript
   } catch {
     // Skip unreadable dirs
   }
   ```
   Acceptable for "skip unreadable dirs" but still no logging.

10. **`src/ui/companion/companion.ts:76,252,381`**
    ```typescript
    } catch {
      // errors handled by companion guard
    }
    ```
    Companion observation errors silently ignored.

## Impact

- **Security:** Network/SSRF errors silently falling back could mask blocked requests
- **Debugging:** Silent failures make it hard to diagnose issues
- **Reliability:** Misconfigurations (e.g., missing shell) hide without any indication

## Recommended Fix

Replace empty catches with at minimum a debug log, or a more informative fallback:

```typescript
// Good pattern:
try {
  // operation
} catch (err) {
  logger.debug('operation failed, falling back', err instanceof Error ? err.message : String(err));
  // explicit fallback
}
```

For truly non-critical paths (e.g., skipping unreadable dirs), at minimum log at trace level.
