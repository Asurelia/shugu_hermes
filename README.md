# Shugu Hermes — Auto-Dev Loop

## Mission
Autonomous code analysis and improvement loop for [Asurelia/Shugu](https://github.com/Asurelia/Shugu).

## How It Works

```
[Daily Cron]
    │
    ▼
Clone latest Asurelia/Shugu
    │
    ▼
Run full test suite + typecheck
    │
    ▼
Deep analysis (bug hunter mode)
    ├── Security (shell injection, permission bypass, etc.)
    ├── Bugs (error handling, edge cases, race conditions)
    ├── Performance (N+1, memory leaks, inefficient algorithms)
    ├── Code quality (duplication, complexity, maintainability)
    └── Architecture (coupling, SOLID violations)
    │
    ▼
For each finding:
    ├── Trivial fix → apply directly + commit
    └── Complex issue → create detailed issue on shugu_hermes
    │
    ▼
Push fixes → run tests → merge if passing
    │
    ▼
Report → Telegram
    │
    ▼
[Loop]
```

## Repository Structure

```
shugu_hermes/
├── README.md                    # This file
├── issues/                      # Structured issue reports
│   └── YYYY-MM-DD/              # Daily reports
│       ├── P0-critical.md
│       ├── P1-high.md
│       └── ...
├── fixes/                        # Applied fixes (committed)
│   └── YYYY-MM-DD/              # Daily fixes
└── reports/                     # Full analysis reports
    └── YYYY-MM-DD/              # Daily full reports
```

## Issue Severity

| Level | Meaning | Action |
|-------|---------|--------|
| P0 | Critical (exploit, crash, data loss) | Fix immediately |
| P1 | High (security, major bug) | Fix if trivial |
| P2 | Medium (non-critical bug, perf) | Create issue |
| P3 | Low (code quality, minor) | Create issue |
| P4 | Enhancement (refactor, optimization) | Create issue |

## Report Format

Each issue/report follows the format in `SKILL.md`.

## Skills

- `auto-dev-loop` — Main orchestrator (load this)
- `shugu-analysis-agent` — Deep code analysis
- `shugu-fix-agent` — Applies fixes

## Contact

Owner: Sylvain (Asurelia)
Claude Code on VPS: 148.230.112.165
