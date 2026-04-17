# Shugu Security Analysis

**Repository:** `https://github.com/Asurelia/Shugu`
**Last Reviewed:** 2026-04-17
**Security Posture:** Hardened (13 issues fixed in 2 recent commits)

---

## Security Architecture

### Defense in Depth

Shugu implements security at multiple layers:

```
Layer 1: Input Sanitization    → src/security.ts, src/meta/config.ts
Layer 2: Permission Classifier → src/policy/classifier.ts
Layer 3: Shell Injection Guard → src/tools/bash.ts (BashTool)
Layer 4: Plugin Sandbox        → src/plugins/host.ts (Docker isolation)
Layer 5: Session Isolation     → BashTool session keys (3-token prefix)
Layer 6: Error Escalation      → Errors propagate, not swallowed
```

---

## Shell Injection Protection

### Two-Layer Detection

**Layer 1 — Config Load Time (`src/meta/config.ts`):**
```typescript
containsShellInjection(input: string): boolean
// Blocks: ; ` $( ` ${ ` || outside of quoted strings
// Called when loading YAML datasets
```

**Layer 2 — BashTool Execution Time (`src/tools/bash.ts`):**
```typescript
// Same containsShellInjection() check at runtime
// Before any shell command execution
```

### What Gets Blocked

```typescript
// BLOCKED patterns:
"; rm -rf /"           // command chaining
"` whoami`"            // backtick expansion
"$(cat /etc/passwd)"   // command substitution
"${HOME}/.ssh"        // variable expansion
"|| echo pwned"       // OR chaining
"&& rm -rf ."         // AND chaining

// ALLOWED (inside quotes):
echo "semicolon; here"     // OK
echo 'backtick` here'      // OK
```

### What Is NOT Blocked (Potential Gaps)

- Inside single-quoted strings: `'${HOME}'` — single quotes stop all expansion in shell
- Piping without chaining: `cat file | grep pattern` — `|` is not blocked
- Redirection: `cmd > file` — `>` not blocked
- Globs: `*.txt` — not typically dangerous

---

## Permission Model

### Deny-by-Default Architecture

All permission contexts default to **deny**:

| Context | Default | Bypass Possible? |
|---------|---------|-----------------|
| `bypass` | deny | only if explicit |
| `plugin_child` | deny | NO (not inherited) |
| `sub_agent` | deny | only with explicit grant |
| `unattended` | deny | via `degradeForUnattended()` |
| `default` | deny | orchestrator capped |

### Permission Degradation for Unattended Mode

When agent runs in `/bg` or `/proactive` mode:

```typescript
degradeForUnattended(policy, mode): Policy
// Reduces capabilities for long-running unattended execution
// Bash tool: session isolation strengthened
// File writes: require explicit paths
// Network: read-only
```

### Commit `06c03d8` — Permission Fixes (S1-S6)

| ID | Issue | Fix |
|----|-------|-----|
| S1 | Loop default-deny was not enforced | `loop.ts` now enforces deny-by-default |
| S2 | Orchestrator could elevate permission | Capped at `'default'` context |
| S3 | Plugin child inherited parent permission | Plugin children now deny-by-default |
| S4 | Bash session key isolation weak | 3-token prefix per session |
| S5 | Session saves swallowed errors | Errors now escalate loudly |
| S6 | SSE stream parse failures logged only | Now escalated with context |

---

## Commit `34efc60` — Security Fixes

| Issue | Severity | Fix |
|-------|----------|-----|
| Shell injection at load time | P0 | `containsShellInjection()` at dataset load |
| Shell injection at execution | P0 | Same check at BashTool runtime |
| EventEmitter listener leak | P1 | Listener detachment on DaemonController exit |
| Unattended permission bypass | P1 | `degradeForUnattended()` enforcement |

---

## Known Security Gaps (Audit Findings)

### Gap 1 — Path Traversal in File Operations

**Not fully mitigated:** User-controlled paths in `path.join()` without sanitization.

```typescript
// POTENTIAL ISSUE:
const filePath = path.join(baseDir, userInput);
// If userInput = "../../../etc/passwd" → path traversal
```

**Status:** No `containsShellInjection()` for paths, only for shell commands.

### Gap 2 — Single-Quote Bypass

Single-quoted strings in shell stop all expansion, but model or user input in single quotes is not checked:

```typescript
// This would NOT be caught:
const cmd = `cat '${userInput}'`;
// If userInput = "'$(whoami)'" → still executes
```

**Status:** Known gap, not yet fixed.

### Gap 3 — Redirection/Piping Not Blocked

`>`, `>>`, `<`, `|`, `2>` are not blocked by `containsShellInjection()`:

```typescript
// This would NOT be blocked:
"cat file | grep secret"     // piped
"echo pwned > /tmp/out"      // redirected
```

**Status:** Could be used for log injection or file write.

### Gap 4 — Plugin Sandbox Escape (Docker only)

Docker sandbox can be disabled via `PCC_DISABLE_DOCKER=1`:
- Sandbox disabled = plugin runs with host privileges
- Only safe if plugin is trusted

**Status:** Intentional opt-out mechanism.

---

## Security Review Checklist (for Auto-Dev Loop)

When auditing Shugu code, check for:

- [ ] All user input → `containsShellInjection()` before shell execution
- [ ] Path operations → sanitized for traversal (`path.resolve()`, `path.normalize()`)
- [ ] Permission checks → no bypass paths through different contexts
- [ ] Error handling → errors escalate, not swallowed
- [ ] EventEmitter → listeners detached on exit
- [ ] Plugin sandbox → Docker enabled by default
- [ ] Session isolation → unique keys per session
- [ ] Empty catch blocks → should have error re-throw or logging

---

## CVSS Risk Ratings (Estimated)

| Issue | CVSS Est. | Rationale |
|-------|-----------|-----------|
| Shell injection (load+exec) | **9.8** (Critical) | RCE if exploitable |
| Permission bypass | **8.1** (High) | Unauthorized actions |
| Listener leak | **5.3** (Medium) | Memory leak, DoS potential |
| Error swallowing | **4.0** (Medium) | Silent failures hide issues |
| Path traversal gap | **7.5** (High) | File read/write if exploitable |
| Single-quote bypass | **6.0** (Medium) | Limited to specific contexts |

---

## Recommendations for Hardening

1. **High Priority:**
   - Add path traversal check alongside shell injection check
   - Block `|`, `>`, `>>`, `<`, `2>` in `containsShellInjection()`
   - Add single-quote content check

2. **Medium Priority:**
   - Enable `typecheck: true` in vitest (catches type safety issues)
   - Add fuzzing tests for shell injection patterns
   - Test plugin sandbox escape scenarios

3. **Low Priority:**
   - Add Content Security Policy for plugin output
   - Add audit logging for all permission-granting events
   - Rate-limit failed authentication attempts
