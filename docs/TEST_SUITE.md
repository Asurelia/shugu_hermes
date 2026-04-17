# Shugu Test Suite

**Repo:** `https://github.com/Asurelia/Shugu`
**Tests:** 947 passing ✅
**Framework:** vitest 4.1.2
**Language:** TypeScript 5.7+

---

## Test Inventory (37 test files)

### Core Engine Tests
| File | Purpose |
|------|---------|
| `agent-depth.test.ts` | Agent reasoning depth limits |
| `agent-teams.test.ts` | Multi-agent team coordination |
| `agent-*.test.ts` | Various agent behaviors |

### Security Tests
| File | Purpose |
|------|---------|
| `security-audit-gaps.test.ts` | Security coverage gaps |
| `security-comprehensive.test.ts` | Full security surface audit |
| `security-utils.test.ts` | Security utility functions |
| `classifier-evasion.test.ts` | Permission classifier evasion attempts |
| `permission-gating.test.ts` | Permission enforcement |
| `permissions.test.ts` | Permission model tests |

### Meta / Harness Tests
| File | Purpose |
|------|---------|
| `meta-shell-injection.test.ts` | Shell injection detection at load AND runtime |
| `meta-proposer-tools.test.ts` | Meta proposer uses only safe tools (no Bash) |
| `meta-config.test.ts` | Config validation |
| `meta-dataset.test.ts` | Meta harness datasets |
| `meta-redact.test.ts` | Output redaction |
| `meta-selector.test.ts` | Task selection |

### Integration Tests
| File | Purpose |
|------|---------|
| `plugin-broker.test.ts` | Plugin broker lifecycle |
| `plugin-brokered-e2e.test.ts` | End-to-end plugin flow |
| `worktree-integration.test.ts` | Worktree operations |
| `work-context.test.ts` | Work context management |

### Feature Tests
| File | Purpose |
|------|---------|
| `batch-command.test.ts` | Batch command execution |
| `budget.test.ts` | Budget management |
| `circuit-breaker.test.ts` | Circuit breaker pattern |
| `credential-domain.test.ts` | Credential domain isolation |
| `daemon-lifecycle.test.ts` | Daemon startup/shutdown, listener cleanup |
| `docker-opt-out.test.ts` | Docker sandbox opt-out (`PCC_DISABLE_DOCKER=1`) |
| `file-tags.test.ts` | File tagging system |
| `highlight.test.ts` | Syntax highlighting |
| `hooks.test.ts` | Hook system |
| `integrations-sanitize.test.ts` | Integration string sanitization |
| `interrupts.test.ts` | Interrupt handling |
| `markdown-loaders.test.ts` | Markdown loader plugins |
| `markdown.test.ts` | Markdown processing |
| `minimax-reasoning.test.ts` | MiniMax reasoning mode |
| `model-routing.test.ts` | Model routing |
| `parsers.test.ts` | Parser functions |
| `scheduler.test.ts` | Task scheduler |
| `session-features.test.ts` | Session features |
| `skills.test.ts` | Skills system |
| `tool-descriptions.test.ts` | Tool description generation |
| `tool-result-pairing.test.ts` | Tool result pairing |
| `tool-router.test.ts` | Tool routing |
| `tools-registry.test.ts` | Tool registry |
| `transport-errors.test.ts` | Transport error handling |
| `trust-store.test.ts` | Trust store |
| `vault.test.ts` | Credential vault |
| `vault-discovery.test.ts` | Vault auto-discovery |
| `verification-agent.test.ts` | Verification agent |
| `workspace.test.ts` | Workspace management |

---

## Running Tests

```bash
cd /home/openclaw/Shugu
npm test              # Run all tests (vitest run)
npm run test:watch   # Watch mode for development
```

### TypeScript Type Checking (currently disabled)

```bash
# In vitest.config.ts:
typecheck: { enabled: false }  # ← DEFAULT
typecheck: { enabled: true }   # ← RECOMMENDED
```

**⚠️ Issue:** `typecheck: { enabled: false }` means TypeScript errors are silently ignored during tests. This should be enabled for production quality.

### Run Specific Test

```bash
npx vitest run tests/meta-shell-injection.test.ts
npx vitest run tests/security-comprehensive.test.ts
```

---

## Test Quality Observations

### Strengths
- ✅ 947 tests covering security, permissions, plugins, meta-harness
- ✅ Dedicated shell injection detection tests (load-time AND runtime)
- ✅ Permission model comprehensively tested (gating, evasion, degradation)
- ✅ Docker opt-out testing
- ✅ Daemon lifecycle with listener cleanup verification
- ✅ Meta-proposer restricted to safe tools only

### Weaknesses / Issues
- ⚠️ `typecheck: { enabled: false }` — TypeScript errors silently ignored
- ⚠️ No test coverage for voice module (`src/voice/`)
- ⚠️ TaskTools persistence untested (in-memory only)
- ⚠️ Session accumulation in `~/.pcc/sessions/` not tested (no TTL/max limit)
- ⚠️ Some tests may be brittle (integration tests that depend on timing)

---

## Key Test Patterns

### Shell Injection Test Pattern
```typescript
// Tests that containsShellInjection() blocks dangerous patterns
// at BOTH dataset load time AND BashTool execution time
test('blocks shell injection at load time', () => {
  const dataset = loadDataset('malicious-prompt-with-semicolons');
  expect(dataset.isValid).toBe(false);
});
```

### Permission Degradation Pattern
```typescript
// Tests that /bg and /proactive modes degrade permissions
test('degradeForUnattended reduces permissions', () => {
  const degraded = degradeForUnattended(originalPolicy, '/bg');
  expect(degraded.canBypass).toBe(false);
});
```

### Docker Opt-Out Pattern
```typescript
test('docker sandbox disabled via env var', () => {
  process.env.PCC_DISABLE_DOCKER = '1';
  const sandbox = createSandbox();
  expect(sandbox.type).toBe('nosandbox');
});
```
