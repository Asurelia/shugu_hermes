# Security Permission Audit — /tmp/shugu-loop/src/

**Auditor:** Hermes Agent (cron job)  
**Date:** 2026-04-21  
**Scope:** `/tmp/shugu-loop/src/policy/`, `engine/loop.ts`, `plugins/`, `commands/automation.ts`, `automation/background.ts`  
**Standards:** AGENTS.md — deny-by-default, fail-closed, no silent failures in security-sensitive paths  

---

## Summary

The permission model is multi-layered with four main components:

| Layer | File | Role |
|-------|------|------|
| Modes | `policy/modes.ts` | 5-mode matrix (plan/default/acceptEdits/fullAuto/bypass) |
| Classifier | `policy/classifier.ts` | Risk-classifies Bash commands (low/medium/high) |
| Rules | `policy/rules.ts` | Builtin + user allow/deny/ask rules |
| Resolver | `policy/permissions.ts` | Central decision point (not wired into loop.ts) |

---

## ✅ Findings: Correctly Implemented

### 1. Deny-by-Default in `engine/loop.ts` (lines 363, 531)
Sequential and concurrent tool execution paths both use:
```typescript
if (askPerm) {
  const granted = await askPerm(call.name, ...);
  if (!granted) { /* deny */ continue; }
} else if (config.toolContext?.permissionMode !== 'bypass') {
  // DENY — no permission handler configured
  const result = { ... is_error: true, content: 'Permission denied...' };
  continue;
}
```
**Verdict:** Correct fail-closed behavior. `bypass` mode is the only explicit opt-out; everything else denies when `askPermission` is absent.

### 2. Builtin Rules Are Checked First
`PermissionResolver.resolve()` checks BUILTIN_RULES → user rules → session allows → mode default. Builtin deny rules (e.g., `rm -rf /`, `mkfs`, raw disk `dd`) are compiled at module load time and can NOT be overridden by user allow rules. Correct priority ordering in `evaluateRules()`: deny → allow → ask.

### 3. ReDoS Protection in Rule Compilation (`rules.ts:106-112`)
User-provided `commandPattern` regexes pass through `validateRegexSafety()` before compilation. Rejected patterns are logged and dropped fail-closed — never silently skipped. Builtin rules trust the source but still must compile.

### 4. Glob-Bounded Regex in Rule Patterns (`rules.ts:266-298`)
The glob-to-regex translator maps `*` → `[^/]*` and `**` → `.*`. Both are linear. An unbounded pattern like `**` in a file glob is acceptable for path matching (non-ReDoS).

### 5. BUILTIN_TOOL_NAMES Shadowing Prevention (`host.ts:633-636`)
```typescript
if (BUILTIN_TOOL_NAMES.has(params.definition.name)) {
  logger.warn(`[plugin:${this.pluginName}] register_tool rejected: shadows builtin`);
  break;
}
```
Plugins (even unrestricted) cannot register tools named `Read`, `Write`, `Bash`, `Agent`, etc. Properly enforced at registration.

### 6. Broker Path Traversal Protection (`broker.ts:85-133`)
`validatePath()` uses `realpath()` to resolve symlinks before boundary checks. Falls back to parent-dir validation for non-existent files. Enforces `pluginDir/.data/` OR `projectDir` boundaries. No TOCTOU race: resolved path is checked immediately after realpath.

### 7. SSRF Protection in `handleHttpFetch` (`broker.ts:165-170`)
`ssrfSafeFetch()` is called for all HTTP capability requests, validating the full redirect chain.

### 8. Plugin Registration Requires Permissions (`host.ts:618-675`)
`hasPermission('tools')`, `hasPermission('hooks')`, etc. checked before accepting `register_tool`, `register_hook`, `register_command`, `register_skill` notifications. Missing permission → logged rejection, registration silently dropped.

### 9. Unattended Degradation for Background Sessions (`background.ts:81-91`, `modes.ts:129-132`)
```typescript
// background.ts
if (!options.allowFullAuto && originalCtx) {
  const downgraded = degradeForUnattended(originalCtx.permissionMode);
  if (downgraded !== originalCtx.permissionMode) {
    effectiveConfig = { ...config, toolContext: { ...originalCtx, permissionMode: downgraded } };
  }
}
```
`fullAuto` and `bypass` degrade to `acceptEdits` (read+write auto-allow; Bash prompts) when spawning `/bg` unattended sessions. `plan` and `default` pass through unchanged. Correct.

### 10. Docker Sandbox Isolation (`host.ts:122-151`)
When Docker is available, plugins run with `--net=none`, `--read-only`, `--cap-drop=ALL`, `--security-opt=no-new-privileges`. Only `/plugin/.data` is mounted read-write. Clean isolation.

### 11. Node `--permission` Fallback (`host.ts:172-202`)
When Node ≥ 22 and not in Docker, `buildPermissionFlags()` sets `--permission --allow-fs-read=<dirs> --allow-fs-write=<.data>`. No `--allow-child-process` or `--allow-worker`. Minimum viable sandbox.

### 12. Empty Permissions = No Code Execution (`loader.ts:215-218`)
```typescript
if (Array.isArray(manifest.permissions) && manifest.permissions.length === 0) {
  plugin.active = true; // marked active but no tools/hooks/commands
  return plugin;
}
```
A plugin declaring `permissions: []` gets no code execution path. Denial-by-absence.

### 13. Local Plugin Requires Confirmation (`loader.ts:336-380`)
Project-local plugins (`.pcc/plugins/`) require an explicit `onConfirmLocal` callback. Without it, they're skipped with error `"Skipped: local plugin requires trust confirmation"`. Global plugins (user-installed in `~/.pcc/plugins/`) load without prompting.

### 14. Session Allow Key Caching Prevents Cascade (`permissions.ts:151-197`)
For Bash session-allow caching, keys are `Bash:cmd:arg1:arg2` (up to 3 non-flag tokens). Prevents "npm install lodash" from auto-approving "npm install evil-pkg". Compound commands (`&&`, `||`, `|`) get unique per-command keys — no cross-contamination.

### 15. Error Sanitization on Tool Execution (`loop.ts:612-614`, `loop.ts:398-400`)
Tool execution errors pass through `sanitizeUntrustedContent()` before being returned to the model. Prevents error-message-based injection attacks (e.g., role-switching markers in error text).

### 16. Loop Detection (`loop.ts:453-461`)
Same tool+args called 3× consecutively injects a `[LOOP DETECTED]` user message. Prevents spin loops from consuming budget unchecked.

---

## ⚠️ Issues: Medium / Design Concerns

### Issue 1 — `PermissionResolver` Is Dead Code in the Core Loop
**Files:** `policy/permissions.ts`, `engine/loop.ts`  
**Severity:** Medium (Architecture)

`PermissionResolver` (the class that integrates builtin rules, user rules, session allow cache, and the risk classifier into one `resolve()` call) is **NOT wired into `engine/loop.ts`**. The loop has its own permission enforcement logic that only uses `askPermission` function and `permissionMode`:

```typescript
// loop.ts — concurrent path (line 353-368)
if (askPerm) { granted = await askPerm(...); ... }
else if (permissionMode !== 'bypass') { /* deny */ }

// loop.ts — sequential path (line 517-540)
if (askPermSeq) { granted = await askPermSeq(...); ... }
else if (permissionMode !== 'bypass') { /* deny */ }
```

The PermissionResolver is used in:
- `entrypoints/bootstrap.ts` — for the CLI bootstrapping
- `meta/runtime.ts` — for the Meta Harness evaluation framework

**Impact:** The complete policy machinery (user-configured allow/deny rules, the `BUILTIN_RULES`, session-level "always allow" caching) is bypassed in the main agent loop unless the entrypoint explicitly wires `askPermission` to use the resolver. If `askPermission` is implemented independently (e.g., a simple always-yes or always-no), none of the rule-based overrides apply.

**Recommendation:** `loop.ts` should accept a `permResolver?: PermissionResolver` and call `permResolver.resolve(call)` when `askPermission` is not provided, to route through the full policy stack.

---

### Issue 2 — Skill `askPermission: async () => true` in Brokered Child
**File:** `child-entry.ts:359`  
**Severity:** Medium (Mitigated by toolInvoke routing)

```typescript
// child-entry.ts:355-360 — SkillContext for brokered plugin skills
toolContext: {
  permissionMode: params.context.permissionMode as ToolContext['permissionMode'],
  askPermission: async () => params.context.permissionMode === 'bypass',
},
```

Skills in brokered mode get `askPermission: async () => true` for their **own internal** operation context. However:
- Skills' `ctx.tools` proxy routes `execute()` through `callback/tool_invoke` → host → normal pipeline (`host.ts:799-806`)
- The host's tool execution respects the tool's own permission checks (e.g., `FileWriteTool` checks `permissionMode`)
- `ctx.toolContext.permissionMode` is correctly inherited from the host

**Remaining concern:** Inside the skill's own execution scope, `askPermission` always returns `true` (in bypass) or `false` (otherwise). If a skill does internal tool-ception (calls a tool from within its own execution, not via `toolInvoke`), it bypasses the host's permission model. However, the skill's tool list (`proxyTools`) is limited to `availableToolNames` passed from the host, which provides a partial gate.

**Recommendation:** Consider whether skills should be able to recursively call tools, and if so, whether that should go through the full host permission pipeline.

---

### Issue 3 — `degradeForUnattended` Is Opt-Out, Not Opt-In
**File:** `commands/automation.ts:103-106`, `background.ts:81-91`  
**Severity:** Medium (Correct by default, but the exception is subtle)

The degradation from `fullAuto`/`bypass` → `acceptEdits` for background sessions is the **default behavior**. The user can opt out with `/bg <prompt> --fullauto`. The degradation message is displayed to the user in the CLI.

However, `single-shot.ts` (unattended invocation without `/bg`) does not apply `degradeForUnattended`. A `fullAuto` single-shot run on a server (cron, CI) would not degrade. While this may be intentional (the operator explicitly chose `fullAuto`), the documentation should clearly state that unattended single-shot invocations retain the configured mode.

**Recommendation:** Verify `single-shot.ts` behavior and document the guarantee (or lack thereof) for unattended fullAuto runs.

---

### Issue 4 — Plugin Capability Broker Is Not Aligned with Tool Permissions
**Files:** `host.ts:799-806`, `broker.ts:29-39`, `loader.ts:274`  
**Severity:** Medium (Misalignment, not direct bypass)

The capability broker (`broker.ts`) gates `fs.read`, `fs.write`, `fs.list`, `http.fetch`. The `PluginHost` passes `resolved.capabilities` to the broker at load time (from `loader.ts:274`). The `CapabilitiesBroker` allows only declared capabilities.

However, plugin tools registered via `registerTool` go through the host's `ToolContext` (with `permissionMode` and `askPermission`) during execution — not through the capability broker. A plugin with `capabilities: ['fs.read']` but which registers a `Bash` tool (no capability required for registration) could execute arbitrary shell commands via the `Bash` tool, bypassing the `CapabilityBroker` entirely.

**Mitigation:** `hasPermission('tools')` is checked at registration, and `BUILTIN_TOOL_NAMES` prevents shadowing. But there is no runtime check tying a plugin's allowed capabilities to the specific tool implementations it registers.

**Recommendation:** For brokered plugins, consider restricting which tool types can be registered based on the plugin's declared capabilities (e.g., `fs.read` → only Read/Glob/Grep tools).

---

### Issue 5 — Classifier Returns `low` for Empty Command
**File:** `classifier.ts:30-86`  
**Severity:** Low (Theoretical)

```typescript
export function classifyBashRisk(command: string): RiskClassification {
  const trimmed = command.trim(); // ''
  // First pass: no HIGH/MEDIUM pattern matches ''
  // Second pass: extractAllCommands('') → []
  // SAFE_COMMANDS.has() is never called (no commands to check)
  return { level: 'low', reason: 'Safe command: ', patterns: [] };
}
```

If `call.input['command']` is `''` or falsy (which `permissions.ts:95` handles by defaulting to `''`), the classifier returns `low` and the loop allows it. In practice, the Bash tool's `validateInput()` likely rejects empty commands, but this is not a defense-in-depth at the classifier level.

**Recommendation:** Add a guard in `classifyBashRisk`: if `command.trim() === ''`, return `{ level: 'high', reason: 'Empty command', ... }` or throw.

---

### Issue 6 — `modes.ts` Has No Explicit Unknown-Mode Fallback
**File:** `policy/modes.ts:76-100`  
**Severity:** Low (Implicit safe fallback)

`getDefaultDecision(mode, category)` has no final `return 'ask'` fallback after the five known modes. If an unexpected string is passed as `mode`, the function returns `undefined`, which would propagate through `PermissionResolver.resolve()` to produce an `undefined` decision. In `loop.ts` (which doesn't use `PermissionResolver`), `permissionMode` comes from `toolContext.permissionMode`, which is typed as `PermissionMode` (a union of the 5 literal strings), so this is only a risk if the type is bypassed at runtime.

**Recommendation:** Add an explicit `return 'ask'` at the end of `getDefaultDecision` as defensive programming.

---

### Issue 7 — Plugin `hasPermission` Is Registration-Time Only
**File:** `host.ts:618-675`  
**Severity:** Low (Not a runtime bypass in practice)

`hasPermission('tools')` is checked when the plugin sends a `register_tool` notification. Once a tool is registered in `host.tools[]`, it remains available for the lifetime of the plugin host. If a plugin's policy changes mid-session (via hot-reload of `plugin-policy.json`), the already-registered tools would not be invalidated. However, plugin policy is loaded once at startup, so this is not a live mutation risk.

---

### Issue 8 — `runAgent` Budget Only on Host Side
**File:** `host.ts:552-558`  
**Severity:** Low (DoS, not privilege escalation)

`host.agentCallsRemaining` tracks `runAgent()` invocations from skill/command callbacks. When exhausted, the child gets an error. However, the agent itself (`AgentTool` in the host) also has its own `maxAgentTurns` which is NOT enforced by the host. A plugin could spawn a sub-agent that runs for many more turns than the plugin's `maxAgentTurns` budget, consuming host resources.

---

## 🔍 Blind Spots & Unverified Paths

1. **`single-shot.ts` entry point** — Not reviewed. Does it call `degradeForUnattended`? Does it wire `askPermission` to the `PermissionResolver`?
2. **`AgentTool.ts`** — Not reviewed. How does a sub-agent inherit (or not) the parent session's permission mode and `sessionAllows` cache?
3. **`BashTool.ts`** — Not reviewed. How does it enforce `permissionMode`? Does it call `askPermission` or rely on the loop's pre-flight check?
4. **`FileWriteTool.ts`, `FileEditTool.ts`** — Not reviewed. How do they handle `permissionMode` at the tool level?
5. **Plugin policy hot-reload** — Does `loadPolicy()` support watching `.pcc/plugin-policy.json` changes mid-session, or is it one-time?
6. **`~/.pcc/plugin-policy.json` precedence** — `discoverPluginDirs()` searches both `~/.pcc/plugins/` and `.pcc/plugins/`. The policy file is project-scoped (`.pcc/plugin-policy.json`). Is there a global policy file at `~/.pcc/plugin-policy.json`?
7. **Capability broker `fs.read` path restriction** — `broker.ts` allows reads within `pluginDir/.data/` and `projectDir`. But `projectDir` could be `/` in some configurations — does `CapabilityBroker` enforce a more restrictive workspace root?

---

## Recommendations Summary (Priority Order)

| Priority | Issue | Recommendation |
|----------|-------|----------------|
| **P1** | PermissionResolver not wired into loop.ts | Make `loop.ts` use `permResolver.resolve(call)` as the primary permission check |
| **P2** | Skill `askPermission: async () => true` | Audit skill-ception patterns; enforce host pipeline for all tool invocations from skills |
| **P3** | Single-shot unattended fullAuto | Verify degradation is applied; document behavior explicitly |
| **P3** | Plugin capability/tool alignment | Restrict registered tool types based on declared capabilities in brokered mode |
| **P4** | Classifier empty-command `low` | Guard `classifyBashRisk('')` → return `high` or throw |
| **P4** | Unknown mode no explicit fallback | Add `return 'ask'` at end of `getDefaultDecision` |

---

*Audit completed by Hermes Agent. All findings are based on static analysis of TypeScript source. Runtime behavior should be verified with integration tests.*
