# [P2] — Performance: ObsidianVault N+1 query patterns

**Files:** `src/context/memory/obsidian.ts`
**Severity:** P2

## Description

Multiple methods in `ObsidianVault` read files **sequentially inside loops** instead of parallel/batch fetching:

| Method | Lines | Issue |
|--------|-------|-------|
| `searchContent()` | 120–153 | `readNote()` called sequentially per note |
| `getStaleNotes()` | 368–390 | `stat()` then `readNote()` sequentially per note |
| `getRecentNotes()` | 454–476 | Same N+1 pattern |
| `getLinkedNotes()` | 192–206 | Nested sequential reads |
| `resolveLink()` | 177–187 | Full vault scan per link resolution |
| `searchByTag()` | 158–172 | Sequential `readNote()` per note |

## Impact

For a vault with 500 notes, a single search can trigger 500+ sequential filesystem syscalls. Link resolution with 10 wikilinks = 10 × (1 listNotes + N reads) = extremely expensive.

## Recommended Fix

Replace sequential `for` loops with `Promise.all` for parallel I/O:
```typescript
const notes = await Promise.all(allNotes.map(p => this.readNote(p)));
```

For `resolveLink()`, build a title→path index once and do O(1) lookups.
