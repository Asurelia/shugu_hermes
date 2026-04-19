# [P2] — Performance: PluginRegistry memory leak — 'crashed' listeners not removed

**File:** `src/plugins/registry.ts`
**Lines:** 93–100, 183–190
**Severity:** P2

## Description

When `loadAll()` registers a 'crashed' listener on a plugin host:
```typescript
host.on('crashed', () => {
  this.unload(plugin.manifest.name);
  this.emit('plugin:crashed', plugin.manifest.name);
});
```

This listener is **never removed** when the plugin is unloaded via `unload()`. Repeated plugin load/unload cycles accumulate stale 'crashed' handlers, causing:
1. Memory leak (listeners hold references to plugin objects)
2. Potential incorrect crash handling for reloaded plugins

## Impact

Repeated load/unload cycles for the same plugin accumulate listeners. The host may still be referenced and its listeners prevent garbage collection.

## Recommended Fix

In `unload()`, call `host.removeListener('crashed', handler)` before shutting down. Store a reference to the handler function for explicit removal:
```typescript
// In loadAll():
const crashHandler = () => { ... };
host.on('crashed', crashHandler);
// Store crashHandler reference on the plugin for later removal

// In unload():
host.removeListener('crashed', plugin._crashHandler);
```
