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
│  │ 3. Detect engine + spawn inner loop             │   │
│  │ 4. Receive summary when inner loop exits        │   │
│  │ 5. Review (test-gated fast path or /magi)       │   │
│  │ 6. Update plan file with findings               │   │
│  │ 7. Auto-commit iteration                        │   │
│  │ 8. If needs human input → AskUserQuestion       │   │
│  │ 9. If complete + no gaps → archive plan         │   │
│  │ 10. Else → goto step 2                          │   │
│  └─────────────────────────────────────────────────┘   │
│                          │                              │
│                    detect engine                         │
│                     ┌────┴────┐                         │
│                     ▼         ▼                         │
│  ┌──────────────────────┐ ┌──────────────────────┐     │
│  │  INNER LOOP (Codex)  │ │ INNER LOOP (Claude)  │     │
│  │     [default]        │ │    [fallback]         │     │
│  │                      │ │                       │     │
│  │ • Builds focused spec│ │ • Receives: plan      │     │
│  │ • Runs codex-adapter │ │   context + focus     │     │
│  │ • Verifies: git diff,│ │ • Implements directly │     │
│  │   build, tests       │ │ • Works until blocked │     │
│  │ • Returns: summary   │ │   OR max 20 turns     │     │
│  │   with engine: codex │ │ • Returns: summary    │     │
│  │                      │ │   with engine: claude  │     │
│  └──────────────────────┘ └──────────────────────┘     │
└─────────────────────────────────────────────────────────┘
```

## Why Nested Loops?

**The Problem**: Long implementation tasks lose context. After 40+ tool calls, Claude starts forgetting earlier decisions, re-reading files it already read, and generally thrashing.

**The Solution**: Break work into chunks. Each inner loop works on one section with fresh context. The outer loop maintains continuity through the plan file.

**Why Codex First?**: Codex is faster for scoped implementation tasks. Ralph already sizes work units to fit one context window with clear goals — a natural match. Using Codex for implementation while Claude handles orchestration and review gives the best of both engines. When Codex isn't available, the original Claude inner loop remains as a fallback.

## Engine Detection

Ralph detects Codex availability once per session and caches the result:

1. Locate `codex-adapter.sh` (from codex-implement skill directory)
2. Check that at least one transport is available:
   - Companion plugin file exists (`codex-companion.mjs`), OR
   - `codex` binary is on PATH

If neither transport is found, ralph logs a warning and uses the Claude inner loop for all iterations. This is a filesystem check, not a live Codex invocation.

**No mid-run fallback**: If Codex runs but produces bad output, the subagent reports it and the outer loop handles retry. Claude fallback only activates for transport-level unavailability detected at session start.

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
   - Detect engine (once per session, cached):
     - codex-adapter.sh exists + transport available → Codex
     - Otherwise → Claude fallback
   - Codex path (default):
     - Task subagent delegates to Codex via codex-adapter.sh
     - Subagent verifies: git diff, build, tests
     - Returns summary with engine: codex
   - Claude path (fallback):
     - Task subagent implements directly
     - Returns summary with engine: claude
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

The inner loop is a Task subagent with limited tool access. Inner loops **cannot spawn their own subagents** and **do not commit** — the outer loop handles both coordination and commits at iteration boundaries.

**Codex engine** (default): The subagent delegates to Codex via `codex-adapter.sh` and verifies the results (git diff, build, tests). See `inner-prompt-codex.md` for the prompt template.

**Claude engine** (fallback): The subagent implements directly using Read, Write, Edit, Bash, Glob, Grep. See `inner-prompt.md` for the prompt template.

Both engines return the same YAML summary format. The `engine` field indicates which ran.
