---
name: session-summary
description: Scan conversation for decisions, bugs, and preferences, then save to memory. Use when wrapping up, or when user says "save", "done", "wrap up", or "session summary".
---

# /memlog:save — Capture Session Knowledge

Synthesize the conversation into durable memory entries. Extracts decisions, bugs, preferences, and approaches — checks against existing memories to avoid duplicates and catch contradictions.

## Prerequisites

1. Resolve the memory directory:
   ```
   CANONICAL_DIR = pwd -P (resolve symlinks)
   SANITIZED_CWD = replace / with - in CANONICAL_DIR
   MEMORY_DIR = ~/.claude/projects/${SANITIZED_CWD}/memory/
   ```
2. If `MEMORY_DIR` doesn't exist, create it: `mkdir -p $MEMORY_DIR`
3. If `MEMORY.md` doesn't exist, create it with this template:
   ```markdown
   # Memory Index

   ## Behavioral Rules (feedback)

   ## Identity (user)

   ## Active Decisions (project)

   ## References (reference)

   ## Sessions (session)

   ## Archived (see ARCHIVE.md for details)
   ```
4. If `ARCHIVE.md` doesn't exist, create an empty one:
   ```markdown
   # Memory Archive

   Superseded and archived memories. Search here when active memories don't have what you need.
   ```
5. Do not ask permission for steps 1-4. Proceed silently.

## Step 1: Scan the Conversation

Extract items in these categories:

**Decisions** — Choices between alternatives, with reasoning (what, why, what was rejected)

**Bugs** — Non-obvious root causes and fixes (skip typos, formatting)

**Preferences** — Corrections ("don't do X") or confirmations ("yes, exactly"). Include the WHY.

**Approaches** — What worked ("Z worked for this case") or failed ("X failed because Y")

**Project knowledge** — External system behaviors, team conventions, deadlines, priorities

## Step 2: Filter

Remove items that are:
- Already in git history (only save the WHY, not the fix itself)
- Trivial (typos, formatting)
- Ephemeral (one-time task details)
- Already in CLAUDE.md

## Step 3: Check Against Existing Memories

Read `MEMORY.md` and for each remaining item:

1. **Already captured?** → Skip. Note as "already in [filename]"
2. **Updates existing memory?** → Flag for edit (not new file)
3. **Contradicts existing memory?** → Run contradiction flow:
   - Show user: "This conflicts with [old memory]. Old says X, new says Y."
   - Ask: "(a) New supersedes old, (b) Old is correct, (c) Both valid"
   - If (a): set old file `status: superseded`, `superseded_by: new.md`, importance → 0.20. Move to Archived in MEMORY.md.
4. **Net new?** → Flag for new memory file

## Step 4: Present Summary for Review

```
## Session Summary — [date]

### Decisions (N items)
1. [Text] → ACTION: [already captured / update X / new memory / contradiction]

### Bugs (N items)
1. [Text] → ACTION: [skip — in git / new session memory]

### Preferences (N items)
1. [Text] → ACTION: [already in X / new feedback memory]

### Approaches (N items)
1. [Text] → ACTION: [skip / new session memory]

### Summary: X new, Y updates, Z already captured, W contradictions
```

Wait for user review. They may approve all, remove items, edit wording, or reclassify.

## Step 5: Execute Saves

For each approved item, use the frontmatter schema in `schema.md`:

**New feedback memory** (preference/correction):
- File: `feedback_[topic].md`
- Content: lead with rule, then **Why:** and **How to apply:** lines
- Set `source: session-summary`, importance = type_weight * source_weight

**New session memory** (decisions, bugs, approaches):
- File: `session_[date]_[topic].md`
- Set `type: session`, `source: session-summary`, `importance: 0.35`
- Content: structured summary, not raw quotes

**New project/user/reference memory**: use appropriate type prefix and weight.

**Update existing memory**: read file, edit relevant section, update `updated` date.

**Handle contradiction**: per user's choice in Step 3.

## Step 6: Update Index

- Add new entries to MEMORY.md in the correct type section
- Include `[i:score]` importance hint
- Move superseded entries to Archived section
- Keep total active entries under 100

## Step 7: Smart Reminders

After saving, check:
- If MEMORY.md has >80 entries: suggest `/memlog:clean`
- If `.last_cleaned` is >28 days old: suggest cleanup
- Scan saved memories for past dates (overdue items): flag them

## Step 8: Context Nudge

After saving:
```
Session knowledge saved to N memory files.
Context is at ~X%. Since decisions are now in memory files,
consider compacting (/compact) to free context space.
```

## Step 9: Create Marker

As the FINAL step, create the session marker:
```bash
touch $MEMORY_DIR/.session_summarized
```
This tells the Stop hook that a summary was saved this session.

## Step 10: Report

```
Session summary complete:
- N new memories created: [filenames]
- N existing updated: [filenames]
- N skipped (already captured or low-value)
- N contradictions resolved
```

## Rules

- NEVER save API keys, tokens, passwords, or credentials. Redact to [REDACTED].
- Never save raw conversation quotes — synthesize into standalone knowledge
- Every memory file MUST have full v2 frontmatter (see schema.md)
- Feedback memories always get Why + How to apply structure
- If fewer than 5 substantive exchanges, skip — nothing worth saving
- Session memories get lower importance (0.35) than feedback (type-dependent)
