# [P2] — Error Handling: Vault writes fail silently in knowledge hook

**File:** `src/plugins/builtin/knowledge-hook.ts`
**Line:** ~53
**Severity:** P2

## Description

Memory hint saves to Obsidian vault use fire-and-forget `.catch()` with zero visibility:
```typescript
}).catch(() => {
  // Silent failure — vault writes are best-effort
});
```

## Impact

If the vault write fails repeatedly (disk full, permission error, path issues), the error is silently discarded. No logging, no retry, no user notification. Repeated failures could indicate a systemic problem with the knowledge system.

## Recommended Fix

At minimum log at `debug` level:
```typescript
}).catch((err) => {
  logger.debug('knowledge-hook: vault write failed', err instanceof Error ? err.message : String(err));
});
```

Consider a counter or periodic warning if failure rate is high.
