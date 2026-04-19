# [P2] — Error Handling: Git commands masked with empty fallbacks in finish-feature/socratic

**Files:** `src/commands/finish-feature.ts`, `src/commands/socratic.ts`
**Lines:** Multiple locations
**Severity:** P2

## Description

Git commands in `finish-feature.ts` and `socratic.ts` use empty string fallbacks on error, making failures indistinguishable from valid empty results:

### `finish-feature.ts`
```typescript
const branch = (await git(['rev-parse', '--abbrev-ref', 'HEAD'], ctx.cwd).catch(() => 'HEAD')).trim();
const status = (await git(['status', '--porcelain'], ctx.cwd).catch(() => '')).trim();
const commitsRaw = await git(['log', `${mergeBase}..HEAD`, '--pretty=%H'], ctx.cwd).catch(() => '');
```

### `socratic.ts`
```typescript
const diff = await git(['diff', `${base}..HEAD`], cwd).catch(() => '');
const commitsRaw = await git(['log', `${base}..HEAD`, '--pretty=%H'], cwd).catch(() => '');
const branch = (await git(['rev-parse', '--abbrev-ref', 'HEAD'], cwd).catch(() => 'HEAD')).trim();
const touched = await git(['diff', `${base}..HEAD`, '--name-only'], cwd).catch(() => '');
```

## Impact

1. `git rev-parse` failure → `'HEAD'` substitutes → protected branch check may incorrectly pass for an unintended branch
2. `git status` failure → empty string → command proceeds as if tree is clean → may merge with uncommitted changes
3. `git log` failure → empty array → "no commits" message shown when failure was the real issue

## Recommended Fix

Catch specific errors, log them, and return an error result rather than silently substituting an empty string that looks like valid output:
```typescript
try {
  const result = await git(['status', '--porcelain'], ctx.cwd);
  return { ok: true, data: result };
} catch (err) {
  logger.warn('finish-feature: git status failed', err instanceof Error ? err.message : String(err));
  return { ok: false, error: 'git status failed — cannot proceed safely' };
}
```
