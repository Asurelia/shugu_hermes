# Shugu Architecture

**Repository:** `https://github.com/Asurelia/Shugu`
**Language:** TypeScript 5.7+
**Runtime:** Node.js >=20.0.0
**Test Runner:** vitest 4.1.2
**Version:** 0.2.0

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                        CLI                               │
│                   (shugu.ts / pcc)                      │
│                         │                                │
│                         ▼                                │
│  ┌─────────────────────────────────────────────────┐    │
│  │              Transport Layer                      │    │
│  │   (SSE client, auto-reconnect, session state)   │    │
│  └─────────────────────────────────────────────────┘    │
│                         │                                │
│                         ▼                                │
│  ┌─────────────────────────────────────────────────┐    │
│  │           Credentials Vault                       │    │
│  │        (File-based, encrypted at rest)           │    │
│  └─────────────────────────────────────────────────┘    │
│                         │                                │
│                         ▼                                │
│  ┌─────────────────────────────────────────────────┐    │
│  │              Engine / Loop                        │    │
│  │        (700 lines, agent orchestration)           │    │
│  │  ┌─────────────────────────────────────────────┐  │    │
│  │  │         Policy Layer                         │  │    │
│  │  │  classifier.ts (deny-by-default)           │  │    │
│  │  │  modes.ts (unattended degradation)          │  │    │
│  │  └─────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────┘    │
│                         │                                │
│                         ▼                                │
│  ┌─────────────────────────────────────────────────┐    │
│  │               Tools Layer                         │    │
│  │  BashTool (session isolation, shell injection)   │    │
│  │  FileTool, EditTool, SearchTool, ...             │    │
│  └─────────────────────────────────────────────────┘    │
│                         │                                │
│                         ▼                                │
│  ┌─────────────────────────────────────────────────┐    │
│  │             Plugin System                        │    │
│  │  host.ts (lifecycle, Docker sandboxing)          │    │
│  │  broker.ts (plugin → tool bridge)               │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

---

## Directory Structure

```
src/
├── entrypoints/
│   └── cli.ts              # CLI entry point
├── engine/
│   └── loop.ts             # Core agent loop (700 lines)
├── transport/
│   ├── client.ts           # SSE client, reconnect
│   └── errors.ts           # Typed transport errors
├── credentials/
│   └── vault.ts            # Encrypted credential store
├── policy/
│   ├── classifier.ts       # Permission classifier
│   └── modes.ts            # Unattended mode degradation
├── tools/
│   ├── bash.ts             # BashTool, session isolation
│   └── *.ts                # Other tools
├── plugins/
│   ├── host.ts             # Plugin lifecycle + Docker
│   └── broker.ts          # Plugin → tool bridge
├── security.ts             # Input sanitization
├── meta/
│   ├── dataset.ts          # Evaluation task datasets
│   ├── config.ts           # Config validation + shell injection
│   ├── evaluator.ts       # Task evaluator (LLM-as-judge)
│   └── proposer.ts         # Improvement proposer
├── voice/
│   └── capture.ts          # Voice module (not wired to CLI)
└── ... (other utilities)
```

---

## Key Design Decisions

### 1. Deny-by-Default Permission Model

Every permission context starts with `deny`. No implicit grants. This means:
- New tools/plugins start locked down
- Explicit permission required for dangerous actions
- Unattended mode gets degraded permissions

### 2. Session-Based Bash Isolation

Each BashTool session gets a unique 3-token prefix key. Sessions cannot:
- Access other sessions' variables
- Inherit permissions from parent
- Bypass the deny-by-default policy

### 3. Two-Layer Shell Injection Detection

Shell injection is caught at **both**:
- Dataset/config load time (preventing malicious configs)
- BashTool execution time (preventing runtime injection)

### 4. Docker Plugin Sandbox (Opt-In)

Plugins run in Docker containers by default for isolation. Can be disabled:
```bash
PCC_DISABLE_DOCKER=1 shugu
```

### 5. Frozen Memory Pattern

Memory is a **snapshot** injected at session start. Changes mid-session are:
- Persisted to disk immediately
- Visible in tool responses immediately
- NOT visible in system prompt until next session

This preserves LLM prefix cache for performance.

---

## Test Coverage (947 tests)

| Category | Files | Focus |
|----------|-------|-------|
| Security | 6 | Shell injection, permissions, classifier evasion |
| Meta/Harness | 6 | Dataset, evaluator, proposer, config |
| Plugins | 2 | Broker lifecycle, e2e |
| Core Engine | 3 | Agent depth, teams |
| Integration | 4 | Worktree, work context |
| Features | 16 | Budget, hooks, markdown, tools, etc. |

---

## What Works Well

1. **Clean separation of concerns** — 14 layers, no circular deps
2. **Security by default** — deny-by-default enforced everywhere
3. **Comprehensive test coverage** — 947 tests, security tested
4. **Meta-harness for quality** — LLM-as-judge evaluation
5. **Frozen memory for perf** — prefix caching preserved

## What Doesn't Work / Missing

1. **Voice module not wired** — `src/voice/` exists but not connected to CLI
2. **TaskTools persistence** — in-memory only, lost on restart
3. **Session accumulation** — `~/.pcc/sessions/` grows forever (no TTL)
4. **vitest typecheck disabled** — TypeScript errors silently ignored
5. **No fuzzing** — shell injection could use fuzzing tests

---

## Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `vitest` | ^4.1.2 | Test runner |
| `typescript` | ^5.7.0 | Type checking |
| `tsx` | ^4.19.0 | TypeScript execution |
| `esbuild` | ^0.27.7 | Bundler |
| `yaml` | ^2.8.0 | YAML parsing (datasets, config) |
| `ink` | ^6.8.0 | React-like CLI output |
| `react` | ^19.2.4 | UI components for CLI |

---

## Key Files (by LoC)

| File | Lines | Purpose |
|------|-------|---------|
| `src/engine/loop.ts` | ~700 | Agent orchestration |
| `src/transport/client.ts` | ~400 | SSE transport |
| `src/tools/bash.ts` | ~350 | Bash execution |
| `src/plugins/host.ts` | ~300 | Plugin lifecycle |
| `src/security.ts` | ~200 | Input sanitization |
| `src/policy/classifier.ts` | ~200 | Permission classification |

---

## Configuration Files

| File | Purpose |
|------|---------|
| `vitest.config.ts` | Test runner config (⚠️ typecheck disabled) |
| `tsconfig.json` | TypeScript config |
| `package.json` | Dependencies, scripts |

### npm Scripts

```bash
npm run dev          # Run CLI in dev mode (tsx)
npm run build        # Build with esbuild
npm run build:tsc    # TypeScript compile
npm run start        # Run built CLI
npm run typecheck    # TypeScript check only
npm test             # Run all tests
npm run test:watch   # Watch mode
npm run benchmark    # Context window benchmark
npm run audit        # Security audit
```
