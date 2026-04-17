# Shugu Meta-Harness System

**Purpose:** Self-evaluation framework for measuring and improving agent quality.

---

## Overview

The meta-harness is Shugu's internal testing/evaluation system. It uses LLMs to judge agent performance, creating a closed feedback loop for agent improvement.

```
┌──────────────────────────────────────────────────────────┐
│                    Meta-Harness Flow                       │
│                                                          │
│  ┌────────────┐    ┌─────────────┐    ┌───────────────┐  │
│  │  Dataset   │───▶│  Evaluator  │───▶│   Proposer    │  │
│  │ (tasks)    │    │ (LLM judge) │    │ (improvements)│  │
│  └────────────┘    └─────────────┘    └───────────────┘  │
│       │                                       │           │
│       │           ┌─────────────┐            │           │
│       └──────────▶│   Config    │◀───────────┘           │
│                   │(validation)  │                        │
│                   └─────────────┘                        │
└──────────────────────────────────────────────────────────┘
```

---

## Components

### 1. Dataset (`src/meta/dataset.ts`)

Task datasets for evaluation. Each dataset contains:
- **Task description** — what the agent should do
- **Evaluation criteria** — how to judge success
- **Command** — optional shell command to verify
- **Expected output** — for automated verification
- **SHA-256 hash** — of agent responses (for reproducible train/holdout split)

**Shell Injection Protection:**
```typescript
// Every dataset is validated at load time:
containsShellInjection(dataset.content)  // blocks ; ` $( ${ ||
```

### 2. Config (`src/meta/config.ts`)

Configuration validation:
- Schema validation for all config keys
- Shell injection check on all string values
- Default value injection
- Type coercion

### 3. Evaluator (`src/meta/evaluator.ts`)

LLM-as-judge evaluation. For each task:

```typescript
interface TaskResult {
  taskId: string;
  prompt: string;
  response: string;
  criteria: string[];
  command?: string;           // Optional verification command
  llmJudgeScore?: number;    // LLM judge score (0-10)
  commandPass?: boolean;     // Command exit code
  executionTimeMs: number;
  error?: string;
}
```

**Scoring Method:**
1. Run agent on task
2. Extract response
3. Ask LLM judge: "Did the agent successfully complete the task?"
4. Optionally run verification command
5. Combine scores into final result

### 4. Proposer (`src/meta/proposer.ts`)

Based on evaluation results, suggests improvements:

```typescript
// Example output:
"Suggested improvement: Add error handling for network timeouts.
The agent failed 3/10 tasks due to unhandled timeout errors.
Recommended fix: Add retry logic with exponential backoff."
```

**Constraint:** The proposer is NOT allowed to use the Bash tool. It can only:
- Read files
- Suggest code changes
- Propose documentation updates
- Recommend test additions

---

## Evaluation Categories

| Category | Description | Example Tasks |
|----------|-------------|---------------|
| `coding` | Code implementation | "Write a function that..." |
| `debug` | Debug existing code | "Fix the bug in..." |
| `refactor` | Improve code quality | "Make this more readable..." |
| `security` | Security audit | "Find vulnerabilities in..." |
| `test` | Write tests | "Add tests for..." |
| `docs` | Documentation | "Document the API..." |

---

## Meta-Shell-Injection Test (`tests/meta-shell-injection.test.ts`)

This test validates the shell injection protection at TWO layers:

```typescript
test('containsShellInjection blocks at load time', () => {
  // Dataset contains: "Contact: $(whoami)"
  const dataset = loadDataset('malicious-dataset.yaml');
  expect(dataset.isValid).toBe(false);  // REJECTED
});

test('containsShellInjection blocks at runtime', () => {
  const result = bashTool.execute("echo $(whoami)");
  expect(result.blocked).toBe(true);  // REJECTED
});
```

**149 lines of shell injection test coverage.**

---

## Meta-Proposer Tools Test (`tests/meta-proposer-tools.test.ts`)

This test ensures the meta-proposer can only use safe tools:

```typescript
test('proposer cannot use Bash tool', () => {
  const tools = proposer.getAllowedTools();
  expect(tools).not.toContain('bash');
  expect(tools).not.toContain('shell');
});
```

**35 lines** — ensures the proposer is sandboxed.

---

## Train/Holdout Split

Datasets use SHA-256 hashes of agent responses for deterministic splits:

```typescript
const hash = sha256(agentResponse);
const split = hash % 10 < 7 ? 'train' : 'holdout';
// 70% train, 30% holdout
// Same response always goes to same split
```

This ensures reproducible evaluation across runs.

---

## Evaluation Metrics

### Per-Task Metrics
- **llmJudgeScore** — 0-10, LLM judge assessment
- **commandPass** — boolean, did verification command succeed
- **executionTimeMs** — how long the task took
- **error** — any error that occurred

### Aggregate Metrics
- **passRate** — fraction of tasks passing
- **avgScore** — mean LLM judge score
- **avgLatency** — mean execution time
- **categoryBreakdown** — performance by evaluation category

---

## Limitations

1. **LLM judge is not perfect** — judge itself can be wrong
2. **No ground truth** — relies on model's own assessment
3. **Verification commands are fragile** — exit codes can be misleading
4. **Eval datasets may not cover all edge cases**
5. **Meta-proposer suggestions need human review**

---

## Integration with Auto-Dev Loop

The auto-dev loop should:
1. Run meta-harness evaluation against latest Shugu code
2. Collect pass rates per category
3. Flag any regressions from baseline
4. If proposer suggests improvements → create GitHub issue
5. Track scores over time to detect drift
