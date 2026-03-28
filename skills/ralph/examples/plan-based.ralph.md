# .ralph.md - Plan-Based Project

This is the original ralph approach: plan files with markdown sections.

## State Files

Plans live in `plans/` directory with this structure:

```yaml
---
status: pending | in_progress | complete | archived
gaps: []
edge_cases: []
progress: []
last_review: null
---

# Plan Title

## Section 1: Description
Implementation details...

## Section 2: Another Part
More details...
```

## Work Units

Each `## ` heading is a work unit (section) to implement.

Work sections in document order unless dependencies require otherwise.
If a section depends on another, complete the dependency first.

## Progress Tracking

Track progress in YAML frontmatter:

- Add completed section names to `progress:` array
- Append discovered issues to `gaps:` array
- Append edge cases to `edge_cases:` array
- Update `last_review:` timestamp after each review

## Review

Use magi synthesis after each section completes.

Magi evaluates:
1. Implementation correctness (tests pass?)
2. Plan alignment (work matches section intent?)
3. Gap discovery (new issues found?)
4. Overall completeness

## Completion

When magi recommends `archive`:
1. Verify `gaps:` array is empty
2. Verify `edge_cases:` array is empty
3. Set `status: archived`
4. Move file to `plans/archived/` subdirectory

If gaps or edge cases remain, continue iterating until resolved.
