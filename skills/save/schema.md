# Memory v2 Frontmatter Schema

Every memory file MUST include this frontmatter:

```yaml
---
name: descriptive-name
description: one-line summary (used to decide relevance — be specific)
type: feedback|user|project|reference|session
created: YYYY-MM-DD
updated: YYYY-MM-DD
importance: 0.XX
status: active|archived|superseded
pinned: false
source: explicit|organic|session-summary
---
```

## Importance Calculation

```
importance = TYPE_WEIGHT * SOURCE_WEIGHT
```

| Type | Weight |
|------|--------|
| feedback | 1.0 |
| user | 0.9 |
| reference | 0.8 |
| project | 0.7 |
| session | 0.5 |

| Source | Weight |
|--------|--------|
| explicit | 1.0 |
| organic | 0.9 |
| session-summary | 0.7 |

## Examples

- Explicit feedback: `1.0 * 1.0 = 1.00`
- Organic feedback: `1.0 * 0.9 = 0.90`
- Session-summary session: `0.5 * 0.7 = 0.35`
- Organic project: `0.7 * 0.9 = 0.63`

## Supersession Fields

When a memory is superseded, add to the OLD file:
```yaml
status: superseded
superseded_by: new_file.md
importance: 0.20
```

Add to the NEW file:
```yaml
supersedes: old_file.md
```

## Content Structure by Type

**feedback**: Lead with the rule, then `**Why:**` and `**How to apply:**` lines.

**user**: Role, goals, expertise, preferences relevant to collaboration.

**project**: Lead with fact/decision, then `**Why:**` and `**How to apply:**` lines.

**reference**: Pointer to external resource with its purpose.

**session**: Structured summary of decisions, bugs, approaches from a session.
