# [P2] — Error Handling: Plugin manifest load swallows all errors silently

**File:** `src/plugins/loader.ts`
**Line:** ~161
**Severity:** P2

## Description

In `loadManifest()`, if a manifest file is malformed JSON or unreadable, the error is caught and swallowed — the function returns `null` with no logging, warning, or user notification. The plugin is silently skipped.

```typescript
} catch {
  return null;
}
```

## Impact

Operators have zero visibility when plugins fail to load due to manifest errors. This could cause critical plugins to be absent without any indication.

## Recommended Fix

Log at `warn` level before returning `null`, including the error message and manifest path:
```typescript
} catch (err) {
  logger.warn(`plugin-loader: failed to load manifest ${manifestPath}`, err instanceof Error ? err.message : String(err));
  return null;
}
```

Also distinguish ENOENT (file not found — normal for optional plugins) from other errors (corruption, permission) which should warn.
