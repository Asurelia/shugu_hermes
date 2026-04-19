# [P1] — Security: SkillContext.askPermission always returns true (permission bypass)

**File:** `src/plugins/child-entry.ts`
**Line:** ~355–360
**Severity:** P1 — Security Bypass

## Description

When a skill invokes a tool via `ctx.toolInvoke()`, the `SkillContext.toolContext.askPermission` is hardcoded to `async () => true`, meaning it **always allows** tool execution regardless of the host's permission mode setting.

```typescript
const skillCtx: SkillContext = {
  // ...
  toolContext: {
    cwd: params.context.cwd,
    abortSignal: new AbortController().signal,
    permissionMode: params.context.permissionMode as ToolContext['permissionMode'],
    askPermission: async () => true,  // ← ALWAYS ALLOWS
  },
  tools: proxyTools,
  // ...
};
```

## Impact

When a skill (which runs as a plugin child) invokes a tool, the tool bypasses the host's permission model. Even in `acceptEdits` or `plan` mode (which should prompt for dangerous operations), skills can execute any tool including `Bash`, `Write`, `Edit`, etc.

**However**, the proxy tools route through `callback/tool_invoke` to the host, which DOES enforce permissions. So the actual bypass is limited to cases where the skill's `toolContext` is used directly.

## Evidence

- `src/plugins/child-entry.ts` lines 355–360 — `askPermission: async () => true`
- Contrast with `buildToolContext()` (lines 157–167) which returns `true` only when `permissionMode === 'bypass'`

## Recommended Fix

Either:
1. Route `askPermission` through the host's permission check via the callback channel, OR
2. Document that skills operate outside the permission model and rely solely on `requiredTools` declarations for access control.

## Related

- Also see: misleading comment in `buildToolContext()` lines 159–162 about "deny by default" when the actual behavior differs.
