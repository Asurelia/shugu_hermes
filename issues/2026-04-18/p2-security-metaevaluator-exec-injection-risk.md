# [P2] — Security: MetaEvaluator uses `exec()` instead of `execFile()` — potential shell injection

**File:** `src/meta/evaluator.ts`
**Line:** 37, 134, 216, 303, 331
**Severity:** P2 (Medium)
**Category:** Security

## Description

The MetaEvaluator uses `execAsync` (based on `child_process.exec`) instead of `execFile` or `spawn` for running task evaluation commands. The `exec` function runs commands through a shell, which means:
1. Environment variables are expanded
2. Shell metacharacters (`;`, `|`, `&&`, etc.) are interpreted
3. The command string is passed to `/bin/sh -c "command"`

While `containsShellInjection()` is called before these exec calls, the sanitization function has known gaps (see `p2-security-containsshellsjection-backslash-gap.md`).

## Evidence

```typescript
// src/meta/evaluator.ts:37
const execAsync = promisify(exec);

// src/meta/evaluator.ts:134
await execAsync(task.setupCommand, { cwd: taskCwd, timeout: 30_000, env: buildSafeEnv() });

// src/meta/evaluator.ts:216
const { stdout } = await execAsync(task.scorer.command, { cwd, timeout: 30_000, env: buildSafeEnv() });

// src/meta/evaluator.ts:303
await execAsync(cmd, { cwd, timeout: 30_000, env: buildSafeEnv() });

// src/meta/evaluator.ts:331
const { stdout } = await execAsync(
  `git -C "${cwd}" ${args.join(' ')}`,  // ← git commands constructed with string interpolation
  { timeout: 30_000, env: buildSafeEnv() }
);
```

The last example at line 331 is particularly concerning — git arguments are joined with spaces and interpolated directly into the shell command string, rather than being passed as an array to `execFile`.

## Impact

If `containsShellInjection()` misses a malicious command (e.g., via backslash-escaped metacharacters), the shell will execute it. Since the meta harness processes YAML datasets from potentially untrusted sources, this could enable RCE.

Note: The `env: buildSafeEnv()` prevents API key leakage, but doesn't prevent command execution.

## Recommended Fix

Replace `execAsync` with a pattern that doesn't invoke a shell:

```typescript
import { execFile } from 'node:child_process';
const execFileAsync = promisify(execFile);

// Instead of:
await execAsync(`git -C "${cwd}" ${args.join(' ')}`, ...);

// Use:
await execFileAsync('git', ['-C', cwd, ...args], { timeout: 30_000, env: buildSafeEnv() });
```

For `setupCommand` and `scorer.command`, use `spawn` with shell disabled, or at minimum pass commands as arrays.
