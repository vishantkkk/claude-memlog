# memlog — Memory System Rules

These rules govern how memories are saved, recalled, and maintained. They supplement the default auto-memory behavior.

## Save Flow (with Contradiction Detection)

When saving a new memory (explicit, organic, or via /memlog:save):

1. **Classify** — Determine type (user/feedback/project/reference/session), name, description.
2. **Check for contradictions** — Scan MEMORY.md for memories of the same type or topic:
   - Read potentially related memory files.
   - Does the new content negate, reverse, or update the existing?
   - Temporal signals: "now", "changed to", "no longer", "switched", "replaced", "instead of".
   - If contradiction: show user both versions. Ask which is current.
     - New supersedes old → set old: `status: superseded`, `superseded_by: new.md`, importance → 0.20, move to Archived.
     - Old is correct → discard new.
     - Both valid → save both with clarifying context.
3. **Write with v2 frontmatter** — see skills/save/schema.md for full spec.
4. **Calculate importance** — `TYPE_WEIGHT * SOURCE_WEIGHT` (feedback=1.0, user=0.9, reference=0.8, project=0.7, session=0.5; explicit=1.0, organic=0.9, session-summary=0.7).
5. **Update MEMORY.md** — correct type section, `[i:score]` hint, keep under 100 active entries.

## Three-Tier Recall

1. **Personal active (MEMORY.md)** — user-specific preferences, identity, workflow. Scan first.
2. **Team active (TEAM_MEMORY.md)** — shared conventions, project decisions, team references. Located at `<project-root>/.claude/memory/TEAM_MEMORY.md`. Check second.
3. **Personal archive (ARCHIVE.md)** — search here when active memories don't answer the question.
4. Never read memory files "just in case." Load only when specifically needed.

## Team Memories

Team memories live in `<project-root>/.claude/memory/` and are checked into git, making them visible to all contributors.

- **What goes in team memory**: repo conventions (feedback type), architectural decisions, project-wide references, shared approach patterns.
- **What stays personal**: user identity, personal preferences, individual workflow, session summaries.
- Team memory files use the same frontmatter schema with an added `scope: team` field.
- The team index is `TEAM_MEMORY.md` (same format as `MEMORY.md`).
- If the project has no `.claude/memory/` directory, team memory is not available — everything saves personal.

## Proactive Surfacing

When the user starts working on a topic, scan BOTH personal `MEMORY.md` and team `TEAM_MEMORY.md` for related memories. If found, surface 2-3 line summaries (not full files). Don't wait to be asked. Keep it to 3-4 lines. If context is already heavy, just mention memories exist — don't load them.

## Auto-Invoke Session Summary

When the user signals end of session ("bye", "done", "wrap up", "closing", "thanks that's it"):
- Auto-invoke /memlog:save before responding with goodbye.
- Skip if: <5 substantive exchanges, already ran this session, or purely informational.

## Supersession Chain

- Old file gets: `status: superseded`, `superseded_by: new.md`
- New file gets: `supersedes: old.md`
- Old entry moves to Archived section in MEMORY.md
- Never delete without user confirmation.

## Security Rules

- NEVER save API keys, tokens, passwords, connection strings, or credentials to memory files.
- If detected in conversation, redact to [REDACTED] before saving.
- Memory files contain decisions, preferences, and context — not secrets.

## Context Budget

The plugin's always-on context (this file + MEMORY.md) should stay under 3%:
- This CLAUDE.md: <100 lines
- MEMORY.md index: <100 entries
- Individual memory files: loaded on-demand only
- After /memlog:save, suggest /compact if context > 35%

## Index Format

MEMORY.md uses type-grouped sections:
```
## Behavioral Rules (feedback)
- [filename.md](filename.md) — description [i:0.99]

## Archived (see ARCHIVE.md for details)
- [old.md](old.md) — ~~description~~ superseded by new.md [i:0.20]
```
