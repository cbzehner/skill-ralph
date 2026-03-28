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
│  │ 5. Run /magi review                             │   │
│  │ 6. Update plan file with findings               │   │
│  │ 7. If needs human input → AskUserQuestion       │   │
│  │ 8. If complete + no gaps → archive plan         │   │
│  │ 9. Else → goto step 2                           │   │
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
1. Magi assesses the plan as "fully realized"
2. AND no gaps remain in the frontmatter
3. AND no edge cases remain in the frontmatter

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
gaps:
  - "Need to handle edge case X"
  - "API error handling not defined"
edge_cases:
  - "What happens when Y is empty?"
progress:
  - section: "Section 1"
    status: complete
  - section: "Section 2"
    status: in_progress
last_review: 2025-01-28T10:00:00Z
---

# Plan Title

## Section 1: Description
...

## Section 2: Description
...
```

## Magi Review

After each inner loop, the outer loop invokes `/magi` to evaluate:

1. **Implementation correctness** - Does the code work? Tests pass?
2. **Plan alignment** - Did the inner loop do what was intended?
3. **Gap discovery** - What new gaps, edge cases, or TODOs emerged?
4. **Completeness assessment** - Is the overall plan now fully realized?

Magi returns:
- Pass/fail on the chunk
- List of newly discovered gaps/edge cases
- Assessment of remaining work
- Recommendation: `continue` | `needs_human_input` | `archive`

If magi is unavailable, the outer loop performs the same evaluation itself (less ideal but functional).

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
   - Parse YAML frontmatter (status, gaps, edge_cases, progress)
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
   - /magi "Review this work: [inner loop summary]
           Against plan: [current section]
           Check: correctness, alignment, gaps, completeness"
   - Parse magi verdict

5. UPDATE PLAN
   - Add any new gaps/edge_cases to frontmatter
   - Update section progress
   - Set last_review timestamp
   - Write plan file

6. ROUTE
   - If magi says "needs_human_input" → AskUserQuestion
   - If magi says "archive" AND no gaps → move to archived/
   - Else → goto step 2
```

## State Persistence

All state lives in the plan file itself (YAML frontmatter). No separate state files needed. This allows:
- Pausing and resuming at any point
- Manual inspection and editing of progress
- Version control of the plan alongside the code
- Multiple sessions working on different plans

## Inner Loop Subagent

The inner loop is a Task subagent with limited tool access (Read, Write, Edit, Bash, Glob, Grep). Inner loops **cannot spawn their own subagents** - if work requires parallel execution, they must return to the outer loop.

See `inner-prompt.md` for the prompt template with substitution variables.
