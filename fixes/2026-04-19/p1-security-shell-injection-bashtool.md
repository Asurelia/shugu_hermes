# Fix: P1 — Shell Injection in BashTool.validateInput()

**Date:** 2026-04-19
**File Modified:** `src/tools/bash/BashTool.ts`

## Issue
`BashTool.validateInput()` only checked that `command` was a non-empty string. It did not check for shell metacharacters, allowing arbitrary shell command construction via `;`, backticks, `$()`, `||`, `&&`, `|`, `>`, `<`.

## Fix Applied
Added shell injection detection to `validateInput()`:

1. Imported `containsShellInjection` from `src/meta/config.js`
2. Added `SHELL_CHAINING_CHARS` regex `/[&|<>]/` for characters not caught by `containsShellInjection`
3. `validateInput()` now returns an error string for:
   - Commands containing `;`, backtick, `$()`, `||` (via `containsShellInjection`)
   - Commands containing `&`, `|`, `<`, `>` (via `SHELL_CHAINING_CHARS`)

## Test Result
All 989 tests pass. No regressions.

## Notes
- `&&`, `|`, `>` etc. are rejected with a message directing users to use separate bash calls
- This is a defense-in-depth fix; the primary protection is the permission mode's `classifyBashRisk` in `fullAuto` mode
- Commands that legitimately need chaining (e.g., `make && make test`) must now be split into separate calls
