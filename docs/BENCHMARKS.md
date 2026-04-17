# Shugu Benchmarks

**Model:** MiniMax-M2.7-highspeed
**Date:** 2026-04-07
**Total Cost:** $0.095 USD

---

## Benchmark A — Context Window Stress Test

**Purpose:** Measure model recall accuracy as context window grows. Tests "needle-in-a-haystack" retrieval.

**Methodology:**
- Inject N facts (city → secret code mappings) into context
- Ask model to retrieve specific facts at different positions (beginning, middle, end)
- Also test: nonexistent entities, multi-fact reasoning
- Accuracy = fraction of correct answers

### Results

| Target Tokens | Facts Injected | Accuracy | Avg Latency | Notes |
|--------------|----------------|----------|-------------|-------|
| 50,000 | 2,000 | **100%** | 4,098ms | Perfect recall |
| 100,000 | 4,000 | **100%** | 3,546ms | Perfect recall |
| 150,000 | 6,000 | **75%** | 4,713ms | Reasoning recall fails |
| 200,000 | 8,000 | **50%** | 7,704ms | Degradation significant |
| 250,000 | 10,000 | **75%** | 4,866ms | Mixed results |
| 300,000 | 12,000 | **0%** | 1,307ms | ⚠️ API ERROR — context limit hit |
| 500,000 | 20,000 | **0%** | 1,958ms | ⚠️ API ERROR |
| 750,000 | 30,000 | **0%** | 2,817ms | ⚠️ API ERROR |
| 1,000,000 | 40,000 | **0%** | 3,796ms | ⚠️ API ERROR |

**API Error:** `"context window exceeds limit (2013)"` — hard limit at ~250K tokens.

### Key Findings

1. **Sweet spot: ≤100K tokens** — 100% accuracy, good latency
2. **150K-250K** — Degraded accuracy, model struggles with reasoning recall
3. **≥300K** — Complete failure (API rejection)
4. **Beginning/middle/end recall** — Beginning and end always work; middle is hardest
5. **Reasoning recall (multi-fact)** — First to degrade as context grows
6. **Hallucination** — Never detected in successful runs

### What This Means for Shugu

- **Safe context limit:** ~100,000 tokens (1.67x safety margin)
- **Danger zone:** >200,000 tokens (50% accuracy, reasoning fails)
- **Hard limit:** 250,000 tokens (API rejects)
- **Compaction recommended when:** context >80,000 tokens

---

## Benchmark B — Obsidian Memory Impact

**Purpose:** Measure how much memory injection improves vs. no-memory baseline.

**Methodology:**
- 10 questions across categories: decision, fact, preference
- Each question asked twice: once WITH memory, once WITHOUT
- Scoring: 0 (wrong/don't know) / 1 (partial) / 2 (correct)
- Latency and token usage tracked

### Results

| Metric | With Memory | Without Memory | Delta |
|--------|-------------|----------------|-------|
| **Avg Score** | **1.8 / 2** | **1.2 / 2** | +50% |
| **Avg Latency** | **4,007ms** | **8,751ms** | -54% (faster) |
| **Avg Output Tokens** | **156** | **439** | -64% (shorter) |
| **Avg Input Tokens** | 215 | 48 | +167 (more context) |

### Per-Question Breakdown

| Question | Category | With Memory | Without Memory |
|----------|----------|-------------|---------------|
| Database choice? | decision | 0 (wrong) | 2 (wrong) |
| Indentation style? | preference | 2 ✓ | 1 (partial) |
| Payment API rate limit? | fact | 2 ✓ | 2 ✓ |
| Deployment host? | decision | 2 ✓ | 1 (partial) |
| Authentication? | decision | 2 ✓ | 0 (wrong) |
| Testing frameworks? | fact | 2 ✓ | 1 (partial) |
| CSS solution? | decision | 2 ✓ | 2 ✓ |
| Caching? | fact | 2 ✓ | 2 ✓ |
| Search technology? | fact | 2 ✓ | 0 (wrong) |
| Feature flags? | fact | 2 ✓ | 1 (partial) |

### Key Findings

1. **Memory provides +50% score improvement** on average
2. **Memory makes model 54% faster** (less generation needed)
3. **Memory reduces output verbosity** by 64% (more concise answers)
4. **Some facts still wrong WITH memory** (database question — memory may have wrong data)
5. **Without memory, hallucinations increase** — model invents plausible answers
6. **Memory particularly helps for:** decisions, preferences, project-specific facts

### What This Means for Shugu

- **Memory is critical** for project-specific knowledge
- **With memory:** faster, more accurate, concise
- **Without memory:** verbose, slower, prone to hallucination
- **For Shugu's memory guardian:** Memory compaction should preserve facts over preferences

---

## Benchmark C — Quick Benchmark Results

**File:** `scripts/benchmark-c-results-1775536857671.json`

Summary (data available in JSON file — full results not yet analyzed).

---

## Scripts Available

```bash
cd /home/openclaw/Shugu

npm run benchmark           # Run context window stress test
npm run audit              # Run security audit script
npm run analyze-traces      # Analyze execution traces
```

### Benchmark Script — `scripts/benchmark-context.ts`

This script runs the context-window stress test (Benchmark A). It:
1. Generates N facts (city → ZULU-XXXX mappings)
2. Asks model to retrieve specific facts
3. Measures accuracy, latency, token usage at each context size

### Audit Script — `scripts/audit.ts`

Security audit script. Runs comprehensive checks:
- Shell injection vectors
- Permission model gaps
- Error handling issues
- Input validation coverage

### Analyze Traces — `scripts/analyze-traces.ts`

Analyzes execution traces from agent runs to identify:
- Token usage patterns
- Loop detection
- Tool call efficiency

---

## Performance Recommendations

### For Shugu Developers

1. **Set context limit warning at 80K tokens** — begin compaction
2. **Hard context limit at 100K tokens** — refuse to proceed if exceeded
3. **Memory is worth the overhead** — enable memory for all project-specific work
4. **Monitor for hallucinations** — when without memory, model invents facts
5. **Test with large codebases** — current benchmarks use synthetic facts

### For Auto-Dev Loop

The benchmark results should inform the loop's analysis:
- Check if Shugu has context management / compaction
- Flag any context sizes approaching the 100K safe limit
- Identify codebases/contexts that would trigger the 250K hard limit
