# Ralph: Architecture

The serious documentation for when things get real.

## Overview

Ralph is a nested loop system:
- **Outer loop**: Orchestrates planning, spawns workers, runs reviews, updates state
- **Inner loop**: Task subagent that implements one section of the plan

```
┌─────────────────────────────────────────────────────────┐
│                     OUTER LOOP                          │
│                 (your Claude Code session)              │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │ 1. Read plan file (YAML frontmatter + markdown) │   │
│  │ 2. Determine next action from plan state        │   │
│  │ 3. Spawn inner loop (Task subagent)             │   │
│  │ 4. Receive summary when inner loop exits        │   │
│  │ 5. Review (test-gated fast path or /magi)       │   │
│  │ 6. Update plan file with findings               │   │
│  │ 7. Auto-commit iteration                        │   │
│  │ 8. If needs human input → AskUserQuestion       │   │
│  │ 9. If complete + no gaps → archive plan         │   │
│  │ 10. Else → goto step 2                          │   │
│  └─────────────────────────────────────────────────┘   │
│                          │                              │
│                          ▼                              │
│  ┌─────────────────────────────────────────────────┐   │
│  │              INNER LOOP (subagent)              │   │
│  │                                                 │   │
│  │  • Receives: plan context + current focus       │   │
│  │  • Works until: blocked OR max 20 turns OR      │   │
│  │    context pressure                             │   │
│  │  • Returns: summary of work done, blockers,     │   │
│  │    any discovered gaps/edge cases               │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## Why Nested Loops?

**The Problem**: Long implementation tasks lose context. After 40+ tool calls, Claude starts forgetting earlier decisions, re-reading files it already read, and generally thrashing.

**The Solution**: Break work into chunks. Each inner loop works on one section with fresh context. The outer loop maintains continuity through the plan file.

## Completion Criteria

The outer loop archives the plan when:
1. Review assesses the plan as "fully realized" (via test-gated fast path or magi)
2. AND no unresolved notes remain in progress entries

## Inner Loop Scope

Each inner loop works until one of:
- **Blocked**: Hits a decision point or blocker requiring outer loop/human input
- **Context pressure**: Losing track of earlier work
- **Turn limit**: Hard limit of 20 turns

The inner loop returns a structured summary including:
- Work completed
- Any blockers encountered
- Newly discovered gaps or edge cases
- Files changed
- Test status

## Plan File Format

YAML frontmatter for machine-readable state, markdown body for human-readable plan:

```markdown
---
status: in_progress  # pending | in_progress | complete | archived
progress:
  - section: "Section 1"
    status: complete
    notes:
      - "gap: API error handling not defined"
      - "edge_case: what happens when Y is empty?"
  - section: "Section 2"
    status: in_progress
    notes: []
last_review: 2025-01-28T10:00:00Z
iterations: 3
no_progress_count: 0
started_at: 2025-01-28T09:00:00Z
---

# Plan Title

## Section 1: Description
...

## Section 2: Description
...
```

## Review

After each inner loop, the outer loop reviews via one of two paths:

**Test-gated fast path**: When the work unit defines tests, the inner loop reports `completed` + `tests_status: passed`, AND the outer loop re-runs those tests and they pass — auto-approve without magi. This cuts the most expensive step from iterations where testing is sufficient gating.

**Full review (magi or self-review)**: When tests aren't defined, didn't pass, or inner loop returned partial/blocked — invoke `/magi` to evaluate correctness, alignment, gap discovery, and completeness.

Both paths produce the same output:
- verdict: pass | fail | needs_work
- gaps_discovered: [list]
- recommendation: `continue` | `needs_human_input` | `archive`

If magi is unavailable, the outer loop performs the evaluation itself using the self-review criteria in SKILL.md.

## Human Input Handling

When magi flags something requiring human decision:
1. Outer loop pauses
2. Uses `AskUserQuestion` to surface the decision
3. Waits for answer
4. Continues with the loop

## Outer Loop Flow (Detailed)

```
/ralph plans/my-feature.md

1. LOAD
   - Read plan file
   - Parse YAML frontmatter (status, progress)
   - Parse markdown body for sections

2. ASSESS
   - If status == "archived" → exit
   - If status == "pending" → set to "in_progress"
   - Identify next section to work on (first non-complete)

3. SPAWN INNER LOOP
   - Task subagent with:
     - Full plan context
     - Current focus section
     - Instructions: "work until blocked or context heavy"
   - Await return

4. REVIEW
   - If tests defined + inner loop completed + outer re-runs tests pass
     → fast path: auto-approve
   - Else → /magi or self-review

5. UPDATE PLAN
   - Update progress entry (status, notes with gaps/edge cases)
   - Set last_review timestamp
   - Write plan file
   - Auto-commit: git add -A && git commit

6. ROUTE
   - If "needs_human_input" → AskUserQuestion
   - If "archive" AND no unresolved notes → move to archived/
   - Else → goto step 2
```

## State Persistence

All state lives in the plan file itself (YAML frontmatter). No separate state files needed. This allows:
- Pausing and resuming at any point
- Manual inspection and editing of progress
- Version control of the plan alongside the code
- Multiple sessions working on different plans

Each iteration auto-commits (`ralph: iteration N - [work unit]`), providing cheap per-iteration rollback via `git revert` without needing per-run directories or separate state files.

## Inner Loop Subagent

The inner loop is a Task subagent with limited tool access (Read, Write, Edit, Bash, Glob, Grep). Inner loops **cannot spawn their own subagents** and **do not commit** — the outer loop handles both coordination and commits at iteration boundaries.

See `inner-prompt.md` for the prompt template with substitution variables.
