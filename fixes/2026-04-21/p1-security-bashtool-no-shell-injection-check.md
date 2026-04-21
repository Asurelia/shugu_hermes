# Shell Injection Audit Report
**Date:** 2026-04-21  
**Scope:** `/tmp/shugu-loop/src/` — TypeScript files  
**Patterns searched:** `exec()`, `spawn()`, `eval()`, `child_process`, `BashTool`, `containsShellInjection()`

---

## Summary

| Status | Count | Files |
|--------|-------|-------|
| ✅ Safe | 8 | GrepTool.ts, discovery.ts, git.ts (utils), git.ts (context), voice/capture.ts, host.ts, REPLTool.ts |
| ⚠️ Conditionally Safe | 3 | BashTool.ts, evaluator.ts, shells.ts |
| ✅ Safe — Hardcoded | 2 | ssh.ts, context/workspace/git.ts |

**No `eval()` usage found.** `exec()` is used in `evaluator.ts` only (defended) and `shells.ts` (hardcoded).

---

## Findings

### 1. `BashTool.ts` — PRIMARY SHELL EXECUTION VECTOR ⚠️
**File:** `src/tools/bash/BashTool.ts`  
**Risk:** HIGH (by design — this is a shell execution tool)

The `BashTool` executes arbitrary user-provided shell commands via `spawn(shell.path, [...shell.args, command])`.

**What it does:**
- User-provided `command` string is passed directly to `shell -c "command"`
- A `bashDenylist` regex check exists (`context.bashDenylist.find()`)
- Uses `buildSafeEnv()` (secrets stripped from env)

**Concerns:**
- **No `containsShellInjection()` validation** — The `BashTool` does NOT call `containsShellInjection()`. Defense is limited to:
  - Denylist regex (bypassable with novel patterns)
  - Docker sandbox (when enabled)
  - Node `--permission` flags (when enabled)
- The denylist check at line 121-129 only blocks patterns pre-configured in `context.bashDenylist`. An attacker who can influence the prompt or tool input could potentially craft commands that bypass specific denylist patterns.
- The `shell.args` includes `-c` on Unix, meaning the entire command string is interpreted by the shell.

**Mitigations present:**
- Docker sandbox (when `isDockerAvailable()` and not `.ts`)
- Node `--permission` flags (Node ≥ 22, not `.ts`)
- Safe env (no API keys/secrets passed)
- Timeout enforcement
- Output truncation

**Recommendation:** Consider adding `containsShellInjection()` as an additional guard before shell execution, especially for non-sandboxed environments.

---

### 2. `evaluator.ts` — Dataset Task Execution ⚠️
**File:** `src/meta/evaluator.ts`  
**Risk:** MEDIUM (dataset-driven, with defenses)

Uses `exec()` from `node:child_process` via `execAsync()` for:
- `task.setupCommand` (line 134)
- `task.scorer.command` (line 216)
- `command_succeeds` criterion (line 303)
- `findRecentFiles` git command (line 332) — **HARDCODED, safe**

**What it does:**
- Commands come from dataset YAML files (task definitions)
- `assertSafeShellCommand()` is called before every `execAsync()`, which calls `containsShellInjection()`
- Uses `buildSafeEnv()` (secrets stripped)

**Concerns:**
- `containsShellInjection()` (in `meta/config.ts`) has documented limitations:
  - Does NOT handle escape sequences
  - Tracks quotes but not backslash escapes
  - **Permits** `&&`, `>`, `>>`, `<`, `|`, `!` (intentional, for legitimate use)
  - Threat model: hostile YAML dataset via clone/pull

**Assessment:** Appropriately defended for dataset task execution. The explicit threat model (hostile dataset) is documented in `meta/config.ts`.

---

### 3. `shells.ts` — Shell Detection ⚠️
**File:** `src/tools/bash/shells.ts`  
**Risk:** LOW (internal use, limited exposure)

Uses `execSync()` with template literals:
```typescript
const result = execSync(`${cmd} ${name}`, { ... });
```
Called by `whichBinary(name)` with hardcoded shell names: `'pwsh'`, `'bash'`, `'zsh'`, `'powershell.exe'`, `'cmd'`.

**Concerns:**
- `process.env['SHELL']` (line 96) is user-controlled and used in `detectUnix()`
- If a malicious `SHELL` env var is set (e.g., `SHELL=/bin/bash; echo pwned`), the path would be checked with `existsSync()` first before being passed to `execSync`

**Assessment:** Low risk. The `envShell` is checked with `existsSync()` before use, and shell names passed to `execSync` are hardcoded. However, the template literal construction is a pattern worth flagging.

---

### 4. `ssh.ts` — Remote Execution ✅
**File:** `src/remote/ssh.ts`  
**Risk:** LOW (properly defended)

Uses `spawn('ssh', [..., '--', command])` with command passed as separate argument (not shell-expanded).

**What it does:**
- `validateSSHCommand()` checks against `SHELL_INJECTION_PATTERN = /[;|&`$(){}!<>\n\r]/`
- `unsafeAllowMetachars` option exists for trusted internal callers only
- Path traversal check on `remotePath` (`..` rejected)

**Assessment:** Well-defended. SSH passes command as argument, not through shell.

---

### 5. `REPLTool.ts` — Node.js Code Execution ✅
**File:** `src/tools/repl/REPLTool.ts`  
**Risk:** LOW (no shell involved)

Uses `spawn('node', ['--input-type=module', '-e', code])`.

**Assessment:** `node -e` executes code directly in Node.js V8 without shell interpretation. The code argument is passed to Node directly, not through a shell. No shell injection risk here.

---

### 6. `GrepTool.ts` ✅
**File:** `src/tools/search/GrepTool.ts`  
**Risk:** LOW

Uses `spawn('rg', args, ...)` with args built from validated pattern and path. Arguments are passed as array elements, not through shell.

---

### 7. `discovery.ts` — CLI Detection ✅
**File:** `src/integrations/discovery.ts`  
**Risk:** LOW

Uses hardcoded detect commands: `'git --version'`, `'node --version'`, etc. Commands are split with `detectCommand.split(/\s+/)` then passed as `[cmd, ...args]` to spawn.

---

### 8. `git.ts` (utils & context) ✅
**Files:** `src/utils/git.ts`, `src/context/workspace/git.ts`  
**Risk:** LOW

Uses hardcoded git arguments: `['rev-parse', '--git-dir']`, `['status', '--short', '--branch']`, etc. No user input in argument construction.

---

### 9. `voice/capture.ts` ✅
**File:** `src/voice/capture.ts`  
**Risk:** LOW

Uses hardcoded commands: `sox rec`, `arecord`, `whisper`, `which`/`where`. Arguments are hardcoded arrays.

---

### 10. `host.ts` — Plugin Sandboxing ✅
**File:** `src/plugins/host.ts`  
**Risk:** LOW (sandboxed)

Uses `spawn()` with internally-built arguments:
- Docker sandbox (full isolation)
- Node `--permission` flags
- Bare process fallback (for dev mode `.ts`)

No user input in spawn arguments. All paths come from plugin manifest/resolved paths.

---

## `containsShellInjection()` Analysis

**File:** `src/meta/config.ts:50`

**Detects:**
- `;` — command chaining
- `` ` `` — backtick substitution
- `$(` — dollar-paren substitution  
- `${` — parameter expansion
- `||` — logical OR chaining

**Deliberately permits:**
- `&&` (short-circuit chain)
- `>`, `>>`, `<` (redirections)
- `|` (pipe)
- `!` (prefix negation)
- Characters inside `'...'` or `"..."` strings

**Limitations (documented):**
- Does NOT handle escape sequences (backslash escapes not tracked)
- Simple quote tracking (no escape handling)
- Not a hardened shell lexer

**Used in:**
- `meta/evaluator.ts` — all dataset-driven command executions
- `meta/dataset.ts` — load-time validation of `setupCommand`, `scorer.command`, `criterion.value`

**Not used in:**
- `BashTool.ts` — user-facing shell execution

---

## No `eval()` Usage Found

No `eval()` calls were found in the TypeScript source. The `exec()` pattern matches in search results were all `RegExp.exec()` or string methods, not `node:child_process exec()`.

---

## Conclusion

The codebase has a **well-understood shell injection surface** centered on `BashTool`. Defenses exist in layers:

1. **BashTool** — Sandbox (Docker/Node perms) + denylist + safe env
2. **Evaluator** — `containsShellInjection()` guard on all dataset-driven commands
3. **Other tools** — Use `spawn()` with array args (no shell) or hardcoded commands

**Key gap:** `BashTool` does not call `containsShellInjection()`. The `bashDenylist` provides some protection but is regex-pattern-based and may be bypassable. Consider adding `containsShellInjection()` as a first-line guard for non-sandboxed BashTool execution.

**Lowest risk paths:** SSH execution, REPL (Node -e), ripgrep spawn, git spawns with hardcoded args.
