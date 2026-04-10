# memlog — Persistent Knowledge for Claude Code

Structured memory system with session summarization, contradiction detection, importance scoring, and two-tier storage. Install once, works on any project.

## What It Does

- **Captures session knowledge** before it's lost to context compaction
- **Detects contradictions** when new info conflicts with saved memories
- **Scores importance** so high-value memories (feedback, preferences) surface first
- **Two-tier storage**: active index (always loaded) + archive (searched on demand)
- **Blocks premature exit** on significant sessions until you save

## Install

```bash
# Add the marketplace
claude plugin marketplace add vishantkk/claude-memlog

# Install the plugin
claude plugin install memlog@claude-memlog-marketplace
```

Or add to your project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "claude-memlog-marketplace": {
      "source": { "source": "github", "repo": "vishantkk/claude-memlog" }
    }
  },
  "enabledPlugins": { "memlog@claude-memlog-marketplace": true }
}
```

## How It Works

1. **During work**: Claude saves memories organically (corrections, decisions, preferences)
2. **End of session**: Run `/memlog:save` (or it auto-invokes when you say "bye"/"done")
3. **The skill scans** the conversation for decisions, bugs, preferences, and approaches
4. **Checks for contradictions** against existing memories before saving
5. **Presents a summary** for your review — you approve, edit, or skip items
6. **Saves to files** with importance scores and structured frontmatter
7. **Stop hook** blocks exit on significant sessions (>8 tool calls) without a summary

## Commands

### `/memlog:save`
Scan conversation, extract knowledge, check contradictions, save to memory. Auto-invoked at session end.

### `/memlog:clean`
Audit all memories for redundancy, staleness, and drift. Proposes KEEP, MERGE, ARCHIVE, or DELETE actions. Run every 2-4 weeks.

## Status Line

A persistent one-liner showing memory count, context usage, and recommended action. Runs after every Claude response so you always know when to save or compact.

**Output format**: `N mems | X% | status`

**Thresholds**:

| Context % | Status | Meaning |
|-----------|--------|---------|
| 0-20% | `fresh` | Work freely, plenty of context remaining |
| 20-35% | `working` | Normal usage, no action needed |
| 35-50% | `save soon` | Consider `/memlog:save` before auto-compaction loses detail |
| 50%+ | `/memlog:save then /compact` | Save memories then compact context now |
| (recently saved) | `saved, /compact` | Save complete, run `/compact` to reclaim context |
| (cleanup overdue) | appends `clean due (Nd)` | Run `/memlog:clean` -- last cleanup was N days ago |

**Example output**:
```
28 mems | 8%  | fresh
28 mems | 22% | working
28 mems | 38% | save soon
28 mems | 55% | /memlog:save then /compact
30 mems | 42% | saved, /compact
45 mems | 12% | fresh | clean due (32d)
```

**Setup**: Plugins cannot auto-register a status line. Add it once to your project or user `settings.json`:

```json
{
  "statusLine": {
    "type": "command",
    "command": "~/.claude/plugins/cache/.../memlog/bin/memlog-status"
  }
}
```

To find the exact path after installing, run:
```bash
find ~/.claude/plugins/cache -name memlog-status -type f 2>/dev/null
```

If you already have a `statusLine` configured, you can pipe both commands together or replace yours with this one.

## Memory Types

| Type | Weight | Use for |
|------|--------|---------|
| feedback | 1.0 | Corrections, preferences, behavioral rules |
| user | 0.9 | Role, expertise, collaboration style |
| reference | 0.8 | Pointers to external systems/resources |
| project | 0.7 | Decisions, architecture, ongoing work |
| session | 0.5 | Conversation summaries |

## Storage

Memories live in Claude Code's standard memory directory:
```
~/.claude/projects/<sanitized-cwd>/memory/
├── MEMORY.md          # Active index (always in context)
├── ARCHIVE.md         # Archived/superseded memories
├── feedback_*.md      # Individual memory files
├── session_*.md
├── .session_summarized  # Marker (created by save skill)
└── .last_cleaned        # Marker (created by clean skill)
```

## FAQ

**Will it conflict with Claude Code's built-in auto-memory?**
No. It coexists. The plugin adds structure (frontmatter, contradiction detection, importance) on top of the standard memory directory.

**What if I already have memories?**
Works with existing files. `/memlog:clean` will inventory them and suggest adding v2 frontmatter.

**How do I share with my team?**
Add the marketplace config to your project's `.claude/settings.json` (checked into git). Every team member gets the plugin on next session.

**What happens if I force-quit without saving?**
The Stop hook tries to block you. If you override it, the hook will re-block next time (the marker is only created by the save skill, not the hook). Some sessions may go unsummarized — that's an accepted tradeoff.

**Does it work on Linux?**
Yes. All shell scripts use portable constructs (`tr` not `sed`, `printf` not `echo`, `find -mmin`).

## License

MIT
