# [P2] — Error Handling: Docker availability check silently swallows errors

**File:** `src/plugins/host.ts`
**Line:** ~95–109
**Severity:** P2

## Description

In `isDockerAvailable()`, if `docker version` fails for ANY reason (not just "not installed"), the error is caught and silently absorbed:

```typescript
} catch {
  _dockerAvailable = false;
}
```

## Impact

If Docker fails due to permission denied, daemon unreachable, or other transient errors, `_dockerAvailable` is incorrectly set to `false`. Plugin features that depend on Docker will silently behave differently than expected.

## Recommended Fix

Log the actual error at `debug` or `info` level. Distinguish ENOENT (Docker not installed — normal) from other failures:
```typescript
} catch (err) {
  if ((err as NodeJS.ErrnoException).code === 'ENOENT') {
    _dockerAvailable = false;
  } else {
    logger.debug('docker availability check failed', err instanceof Error ? err.message : String(err));
    _dockerAvailable = false;
  }
}
```
