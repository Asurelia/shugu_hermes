# [P3] — Code Quality: Multiple files exceed 500 lines — God object risk

**File:** Multiple
**Line:** Various
**Severity:** P3 (Low)
**Category:** Code Quality

## Description

Several source files exceed 500 lines, indicating potential "god object" anti-patterns where a single file/module has too many responsibilities. Large files are harder to test, review, and maintain.

## Evidence

| File | Lines | Concerns |
|------|-------|----------|
| `src/plugins/host.ts` | 858 | Plugin lifecycle, Docker sandbox, IPC proxy — too many concerns |
| `src/entrypoints/repl.ts` | 728 | REPL main entry, command dispatch, UI rendering |
| `src/engine/loop.ts` | 717 | Agentic loop — already well-structured but large |
| `src/context/memory/obsidian.ts` | 707 | Obsidian vault integration — complex |
| `src/meta/cli.ts` | 640 | CLI commands, arg parsing |
| `src/agents/orchestrator.ts` | 522 | Agent spawning, delegation |
| `src/context/memory/agent.ts` | 506 | Agent memory management |
| `src/automation/daemon.ts` | 465 | Daemon controller — approaching threshold |
| `src/plugins/loader.ts` | 434 | Plugin loading |
| `src/meta/evaluator.ts` | 431 | Meta harness evaluation |

## Impact

- Harder to test individual functions
- Harder to reason about side effects
- Single-responsibility principle violations
- Longer compile times (TypeScript)

## Recommended Fix

For files approaching 700+ lines, consider splitting:
- `src/plugins/host.ts` → extract Docker sandbox logic, IPC proxy, permission handling into separate modules
- `src/entrypoints/repl.ts` → extract UI rendering, command handlers
- `src/meta/cli.ts` → extract command implementations into separate files

Files at 500-600 lines are acceptable if well-structured (like `loop.ts`).
