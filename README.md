# Shugu Hermes — Autonomous Dev Loop for Asurelia/Shugu

**Target Repository:** [Asurelia/Shugu](https://github.com/Asurelia/Shugu)
**Documentation Repository:** [Asurelia/shugu_hermes](https://github.com/Asurelia/shugu_hermes)

---

## Project Status

| Metric | Value | Notes |
|--------|-------|-------|
| Test Count | 947 passing | vitest 4.1.2 |
| TypeScript | ⚠️ Disabled in tests | `vitest.config.ts` has `typecheck: { enabled: false }` |
| Security | ✅ Hardened | 13 issues fixed in commits `06c03d8` + `34efc60` |
| Architecture | ✅ Clean | 14 layers, no circular deps |
| Context Limit (safe) | ~100K tokens | 100% recall accuracy |
| Context Limit (hard) | 250K tokens | API rejection |
| Memory Impact | +50% score | vs no-memory baseline |

---

## Documentation

| Document | Contents |
|----------|----------|
| [README.md](./README.md) | This file — project overview |
| [docs/TEST_SUITE.md](./docs/TEST_SUITE.md) | 37 test files, how to run, quality assessment |
| [docs/BENCHMARKS.md](./docs/BENCHMARKS.md) | Context window stress test, memory impact study |
| [docs/SECURITY.md](./docs/SECURITY.md) | Shell injection, permissions, known gaps |
| [docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md) | 14-layer architecture, key files, design decisions |
| [docs/META_HARNESS.md](./docs/META_HARNESS.md) | Self-evaluation framework, LLM-as-judge |

---

## What Is Shugu?

Shugu is an autonomous AI coding agent by [Asurelia](https://github.com/Asurelia), built for MiniMax M2.7.

- **Version:** 0.2.0
- **Language:** TypeScript 5.7+
- **Runtime:** Node.js >=20.0.0
- **CLI Name:** `pcc` / `shugu`
- **Owner:** Sylvain (Asurelia / Rafai)

### Key Capabilities

- Agent orchestration with permission-based security
- BashTool with session isolation
- Plugin system with Docker sandboxing
- Meta-harness for self-evaluation
- Frozen memory pattern for prefix caching

---

## Current Security Posture

**Status:** Hardened — 13 critical/high issues resolved in recent commits.

### Shell Injection: 2-Layer Protection
1. Config load time — `src/meta/config.ts` `containsShellInjection()`
2. BashTool execution time — `src/tools/bash.ts` same check

### Permission Model: Deny-by-Default
- All contexts (bypass, plugin_child, sub_agent, unattended) → deny
- Unattended degradation via `degradeForUnattended()`
- Bash session isolation via 3-token prefix keys

### Known Gaps
- Path traversal not checked (only shell injection)
- Single-quote bypass not blocked
- Redirection (`|`, `>`, `>>`) not blocked
- Docker sandbox opt-out (`PCC_DISABLE_DOCKER=1`)

---

## Benchmark Results Summary

### Context Window (MiniMax-M2.7-highspeed)

| Context Size | Accuracy | Latency | Status |
|--------------|----------|---------|--------|
| 50K-100K tokens | 100% | 3.5-4.1s | ✅ Optimal |
| 150K-250K tokens | 50-75% | 4.7-7.7s | ⚠️ Degraded |
| 300K+ tokens | 0% (API error) | ~2-4s | ❌ Hard limit |

**Recommendation:** Keep context <100K tokens (1.67x safety margin). Hard limit at ~250K.

### Memory Impact

| Mode | Avg Score | Avg Latency | Output Tokens |
|------|-----------|-------------|---------------|
| With Memory | 1.8/2 | 4,007ms | 156 |
| Without Memory | 1.2/2 | 8,751ms | 439 |
| **Improvement** | **+50%** | **-54%** | **-64%** |

**Memory makes the model:** faster, more accurate, more concise.

---

## Auto-Dev Loop

This repository powers an autonomous dev loop that runs daily at 08:00 UTC.

### Loop Process

```
[08:00 Daily]
    │
    ▼
Clone latest Asurelia/Shugu
    │
    ▼
npm test + typecheck
    │
    ▼
Deep analysis:
├── Security (shell injection, permissions)
├── Bugs (error handling, edge cases)
├── Performance (memory leaks, N+1)
├── Code quality (duplication, complexity)
└── Architecture (coupling, SOLID)
    │
    ▼
For each finding:
├── P0/P1 → Apply fix + commit to shugu_hermes
└── P2-P4 → Create structured issue
    │
    ▼
Push to github.com/Asurelia/shugu_hermes
    │
    ▼
Report to Telegram
```

### Skills Used

| Skill | Purpose |
|-------|---------|
| `auto-dev-loop` | Main orchestrator |
| `shugu-analysis-agent` | Deep code analysis |
| `shugu-fix-agent` | Applies fixes |
| `memory-guardian` | Memory consolidation |

---

## Issue Severity Scale

| Level | Meaning | Action |
|-------|---------|--------|
| P0 | Critical — RCE, data loss, exploit | Fix immediately |
| P1 | High — security bypass, major bug | Fix if trivial |
| P2 | Medium — non-critical bug, perf | Create issue |
| P3 | Low — code quality, minor | Create issue |
| P4 | Enhancement — refactor, optimization | Create issue |

---

## Repository Structure

```
shugu_hermes/
├── README.md                    # This file
├── docs/
│   ├── TEST_SUITE.md            # Test documentation
│   ├── BENCHMARKS.md            # Benchmark results
│   ├── SECURITY.md              # Security analysis
│   ├── ARCHITECTURE.md          # Architecture overview
│   └── META_HARNESS.md          # Meta-harness system
├── issues/                      # Daily issue reports
│   └── YYYY-MM-DD/
└── fixes/                       # Applied fixes
    └── YYYY-MM-DD/
```

---

## How to Contribute

1. The auto-dev loop runs daily and creates issues automatically
2. Issues are structured in `issues/YYYY-MM-DD/`
3. Each issue includes: file, line, severity, description, impact, fix
4. P0/P1 fixes are applied directly and committed
5. P2-P4 issues are documented for human review

---

## Contact

- **Owner:** Sylvain (Asurelia / Rafai)
- **Repository:** https://github.com/Asurelia/shugu_hermes
- **Target:** https://github.com/Asurelia/Shugu
- **VPS:** 148.230.112.165 (spoukie.uk)

---

## Last Updated

- **Date:** 2026-04-17
- **Test Count:** 947 (was 599 before 2026-04-17 merge)
- **Commits Analyzed:** `06c03d8`, `34efc60`, `chore/rodin-reported-items-20260417`
- **Skills Created:** `auto-dev-loop`, `shugu-analysis-agent`, `shugu-fix-agent`, `memory-guardian`, `shugu-project`, `shugu-workflow`
