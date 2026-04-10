---
name: consolidate
description: Audit all memory files for redundancy, staleness, and drift. Proposes merge, archive, or delete actions for user approval.
disable-model-invocation: true
---

# /memlog:clean — Audit and Consolidate Memories

Periodic maintenance for the memory system. Inventories all memories, recalculates importance, proposes actions, and executes after user approval.

Use when: memory index feels cluttered, user says "clean up memories", or every 2-4 weeks.

## Prerequisites

1. Resolve memory directory (same as /memlog:save):
   ```
   CANONICAL_DIR = pwd -P
   SANITIZED_CWD = replace / with - in CANONICAL_DIR
   MEMORY_DIR = ~/.claude/projects/${SANITIZED_CWD}/memory/
   ```
2. If `MEMORY_DIR` or `MEMORY.md` doesn't exist, tell user: "No memories found. Use /memlog:save first."

## Phase 1: Inventory

Read every `.md` file in the memory directory. For each file:

1. **Recalculate importance**: `TYPE_WEIGHT * SOURCE_WEIGHT` (see save/schema.md)
2. **Content overlap**: scan descriptions in same type group, flag pairs covering the same topic
3. **Staleness signals**:
   - Dates in the past (e.g., "pending approval 2026-03-30" when today is later)
   - "TODO" or "needs verification" markers
   - References to files/functions that may no longer exist
   - `status: superseded` entries still in MEMORY.md active section
4. **Size**: flag files >200 lines
5. **Tier correctness**: is the memory in the right index (MEMORY.md vs ARCHIVE.md)?

## Phase 2: Propose Actions

For each memory, recommend ONE action:

| Action | Criteria | What happens |
|--------|----------|-------------|
| **KEEP** | Relevant, correct, right tier | No changes |
| **MERGE** | 2+ memories overlap on same topic | Synthesize into one, archive originals |
| **ARCHIVE** | Low importance, stale, not pinned | Move to ARCHIVE.md, set `status: archived` |
| **DELETE** | Fully superseded AND archived, or user confirms irrelevant | Remove file and index entry |

Present as a table:

```
## Consolidation Plan — [date]

### Inventory: N total (X active, Y archived)

| # | Memory | Current [i] | New [i] | Action | Reason |
|---|--------|-------------|---------|--------|--------|
| 1 | feedback_no_polling.md | 0.99 | 0.90 | KEEP | Active, relevant |
| 2 | decisions_old.md | 0.70 | 0.70 | ARCHIVE | 45 days stale, no recent access |
| 3 | feedback_a.md + feedback_b.md | 0.90 | 0.90 | MERGE | Same topic |
| 4 | session_old.md | 0.20 | — | DELETE | Superseded, in archive |

### Summary
- KEEP: N | MERGE: N | ARCHIVE: N | DELETE: N

Proceed? [y/n/select specific items]
```

**Wait for user approval before any changes.**

## Phase 3: Execute

**MERGE:**
1. Read both files fully
2. Synthesize into one, preserving all unique information
3. Keep the more descriptive filename
4. Set `supersedes: [merged files]` in new frontmatter
5. Archive originals (don't delete)
6. Update MEMORY.md

**ARCHIVE:**
1. Move entry from MEMORY.md to ARCHIVE.md
2. Set `status: archived` in frontmatter
3. Keep file intact

**DELETE:**
1. Confirm with user one final time
2. Remove file from disk
3. Remove from MEMORY.md and ARCHIVE.md
4. Clean up `supersedes`/`superseded_by` references in other files

## Phase 4: Rebuild Indexes

1. Rewrite MEMORY.md — grouped by type, sorted by importance
2. Rewrite ARCHIVE.md with updated entries
3. Verify MEMORY.md active entries <= 100
4. Check for orphans (files not in any index, or index refs to missing files)

## Phase 5: Write Cleanup Marker

```bash
touch $MEMORY_DIR/.last_cleaned
```

## Phase 6: Report

```
## Consolidation Complete — [date]

### Actions Taken
- Merged: [list]
- Archived: [list]
- Deleted: [list]

### Health
- Active memories: N (was M)
- Archived memories: N (was M)
- Index lines: N/100
- Next cleanup recommended: [date + 4 weeks]
```

## Rules

- Never delete without explicit user confirmation
- Never merge memories of different types
- Pinned memories are NEVER auto-archived
- Feedback memories have floor importance of 0.50 — stay active unless user explicitly archives
- NEVER save API keys, tokens, passwords, or credentials in any memory file
