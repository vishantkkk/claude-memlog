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

Read `MEMORY.md` (and `TEAM_MEMORY.md` if it exists) and for each remaining item:

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

## Step 5b: Team Memory Routing

Before writing files, determine where each approved item should be saved:

1. **Check for team memory directory**: Does `<project-root>/.claude/memory/` exist?
   - If NO → skip this step entirely. Save everything to personal `MEMORY_DIR` (default behavior).
   - If YES → proceed to step 2.

2. **For each item, ask the user**: "Is this personal or team-wide?"
   - Present items grouped: "These look team-relevant: [list]. These look personal: [list]. Adjust?"
   - Good candidates for **team**: feedback about repo conventions, project decisions, reference info, approach patterns that apply to all contributors.
   - Good candidates for **personal**: user identity, personal preferences, individual workflow choices.

3. **Route items**:
   - **Personal** → save to `MEMORY_DIR` (existing behavior, `~/.claude/projects/<sanitized-cwd>/memory/`)
   - **Team** → save to `<project-root>/.claude/memory/`
     - If `TEAM_MEMORY.md` doesn't exist, create it with this template:
       ```markdown
       # Team Memory Index

       ## Behavioral Rules (feedback)

       ## Active Decisions (project)

       ## References (reference)

       ## Sessions (session)

       ## Archived (see ARCHIVE.md for details)
       ```
     - Use the same frontmatter schema (schema.md) for team memory files
     - Add an extra frontmatter field: `scope: team`
     - Update `TEAM_MEMORY.md` index (same format as MEMORY.md, with `[i:score]` hints)

4. **Naming**: Team memory files use the same naming convention (e.g., `feedback_[topic].md`, `session_[date]_[topic].md`) but are stored in the project `.claude/memory/` directory.

## Step 6: Pattern Detection (Auto-Promote)

After saving session memories, check if any extracted preference or correction is a recurring pattern:

1. **Scan session history** — For each new preference/correction from Step 1, search `session_*.md` files in `$MEMORY_DIR` for matching topics. Match on:
   - Same file or function names mentioned
   - Same behavioral pattern (e.g., "don't use X", "always do Y")
   - Same tool, library, or config references
   Use simple keyword overlap — not semantic similarity.

2. **Count occurrences** — If the same correction appears in **2 or more previous** session memories (excluding the current session):
   ```
   Pattern detected: "[correction summary]" has come up N times across sessions.
   Previous occurrences: session_2026-03-15_topic.md, session_2026-04-01_topic.md
   Promote to permanent feedback rule? (y/n)
   ```

3. **If approved** — Create `feedback_[topic].md` with:
   - `type: feedback`, `source: organic`, importance = 0.90
   - Content: synthesized rule with **Why:** and **How to apply:**
   - Set the source session memories to `status: superseded`, `superseded_by: feedback_[topic].md`, importance → 0.20
   - Move superseded entries to Archived section in MEMORY.md

4. **If declined** — Save as normal session memory, no promotion.

## Step 7: Update Index

- Add new entries to the appropriate index:
  - Personal items → `MEMORY.md` in `MEMORY_DIR`
  - Team items → `TEAM_MEMORY.md` in `<project-root>/.claude/memory/`
- Include `[i:score]` importance hint
- Move superseded entries to Archived section
- Keep total active entries under 100 per index

## Step 8: Smart Reminders

After saving, check:
- If MEMORY.md has >80 entries: suggest `/memlog:clean`
- If `.last_cleaned` is >28 days old: suggest cleanup
- Scan saved memories for past dates (overdue items): flag them

## Step 9: Context Nudge

After saving:
```
Session knowledge saved to N memory files.
Context is at ~X%. Since decisions are now in memory files,
consider compacting (/compact) to free context space.
```

## Step 10: Create Marker

As the FINAL step, create the session marker:
```bash
touch $MEMORY_DIR/.session_summarized
```
This tells the Stop hook that a summary was saved this session.

## Step 11: Report

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
