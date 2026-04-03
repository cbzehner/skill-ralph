---
name: ralph
description: Iterative implementation loop with review checkpoints. Use for multi-step tasks that benefit from chunked execution and verification. Use this whenever the user has a plan file, multi-section task, or work that should be executed systematically with reviews between chunks.
argument-hint: "[state-file]"
effort: high
allowed-tools: Task, Read, Write, Edit, Glob, Grep, Skill, AskUserQuestion, Bash
---

# Ralph: Iterative Implementation Loop

Execute work through iterative cycles with review checkpoints between chunks.

## Invocation

```
/ralph [state-file]
```

Examples:
- `/ralph plans/my-feature.md` - Execute a plan file
- `/ralph` - Auto-detect state file per project guidance

## When to Use

- Multi-step implementation tasks (3+ work units)
- Plans with clear acceptance criteria or sections
- Tasks benefiting from review checkpoints between chunks
- Work that may span multiple context windows

## When NOT to Use

- **Simple single-step tasks**: One function, one bug fix—just do it directly
- **Exploration/research tasks**: Use Task with `subagent_type: Explore` instead
- **Tasks needing parallel execution**: Inner loops run sequentially; parallel coordination adds context burden without speedup
- **Unclear requirements**: Clarify first, then plan, then ralph
- **Tasks estimated >60 minutes**: Split into separate plans; long sessions hit context exhaustion

## Host Adaptation

Ralph's inner loop uses Claude Code primitives (Task subagents, Skill tool,
AskUserQuestion). On hosts without these:

- **No Task tool**: Run work units sequentially in the current session instead
  of spawning subagents. Skip `max_turns` enforcement — use the guardrail
  timers instead.
- **No Skill tool**: Use self-review (Step 4 fallback) instead of `/magi`.
  The self-review criteria already exist in the skill.
- **No AskUserQuestion**: Make conservative assumptions and document them.
  When a decision truly requires human input, state what you'd recommend and
  why, then pause and wait for the user to respond.
- **`${CLAUDE_SKILL_DIR}` unavailable**: Resolve file references relative to
  the skill's actual directory path.

The outer loop (LOAD → ASSESS → WORK → REVIEW → UPDATE → ROUTE) is the same
regardless of host. Only the execution primitives change.

## Guardrails

Ralph enforces limits to prevent context exhaustion:

| Threshold | Action |
|-----------|--------|
| 30 minutes | Log warning, suggest checkpoint |
| 45 minutes | Recommend breaking to new session |
| 60 minutes | Force checkpoint, prompt user to split plan |
| 3 consecutive no-progress iterations | Circuit breaker: escalate to user |
| Inner loop max turns (20) | Force exit, return partial summary |

Track iteration progress in state file frontmatter:
```yaml
iterations: 0
no_progress_count: 0
started_at: 2026-01-30T10:00:00Z
```

## Project Guidance

Ralph adapts to project-specific conventions via a `.ralph.md` file.

### Finding .ralph.md

Search in order:
1. **Git repository root**: `git rev-parse --show-toplevel` then check for `.ralph.md`
2. **Walk up from current directory**: Check each parent until `.ralph.md` found or root reached
3. **State file's directory**: If state file provided, check its directory

```bash
# Get git project root
git rev-parse --show-toplevel 2>/dev/null
```

### If No .ralph.md Found

Use AskUserQuestion to offer creating one:

"No `.ralph.md` found for this project. Would you like to create one?"
- **Yes, help me create it** - Ask design questions, generate .ralph.md
- **Use defaults** - Continue with plan-file conventions
- **Skip for now** - Continue without guidance

See `examples/` in this skill's directory for templates and the README for guidance on crafting .ralph.md files.

## The Loop

### 1. LOAD

1. Find and load `.ralph.md` if present (provides project-specific guidance)
2. Read the state file
3. Parse state according to guidance (or use default plan format)

**Default format** (when no .ralph.md):
```yaml
---
status: pending  # pending | in_progress | complete | archived
progress: []     # each entry: { section, status, notes: [] }
last_review: null
iterations: 0
no_progress_count: 0
started_at: null  # Set to current timestamp on first ASSESS
---

# Title

## Section 1
...
```

Progress entries track gaps and edge cases per work unit:
```yaml
progress:
  - section: "Section 1"
    status: complete
    notes:
      - "gap: API error handling not defined"
      - "edge_case: empty input returns null"
  - section: "Section 2"
    status: in_progress
    notes: []
```

If file lacks frontmatter, add defaults.

### 2. ASSESS

- Check completion status → if complete/archived, inform user and exit
- Update status to `in_progress` if pending
- **Initialize guardrails** (first iteration only):
  - If `started_at` is null, set to current timestamp (run `date -Iseconds`)
  - If `iterations` is missing, set to 0
- **Check guardrails**:
  - Compare current time to `started_at` → warn at 30min, recommend break at 45min, force checkpoint at 60min
  - Check `no_progress_count` → escalate to user if >= 3
- Identify work units per guidance:
  - **Default**: `## ` headings not in progress array
  - **Per guidance**: acceptance criteria, issues, custom sections
- Group related units if guidance suggests logical groupings
- If all units complete → proceed to final review

**Work unit prioritization:**
1. Resume any `status: partial` unit from previous iteration
2. Pick first incomplete unit in document order
3. If a unit depends on another, complete the dependency first

**Atomic sizing:** Each work unit should fit in one context window. If a unit seems too large (multiple files, complex logic), split it before starting.

### 3. SPAWN INNER LOOP

Use the Task tool to spawn a subagent:

```
Task(
  subagent_type: "general-purpose",
  description: "Implement: [work unit summary]",
  max_turns: 20,
  prompt: [see ${CLAUDE_SKILL_DIR}/inner-prompt.md, include project guidance if present]
)
```

Inner loop works until:
- Work unit complete (sets `status: completed` in return summary)
- Blocked (needs decision, unclear requirement)
- Max turns reached (hard limit: 20)
- Context pressure (losing track of earlier work)

**Subagent limitation:** Inner loops cannot spawn their own subagents. If work requires parallel execution, return to outer loop and let it coordinate.

Returns structured summary (see inner-prompt.md for format).

### 4. REVIEW

Review has two paths: a fast path when tests gate quality, and the full magi path otherwise.

#### Test-gated fast path

Use this when ALL of these are true:
1. The work unit or `.ralph.md` defines test commands or test criteria
2. Inner loop returned `status: completed` AND `tests_status: passed`
3. **Outer loop re-runs the tests itself and they pass** (trust but verify)

When the fast path applies:
- verdict: `pass`
- recommendation: `continue` (or `archive` if all units complete)
- Note any `gaps_discovered` or `edge_cases_discovered` from the inner loop summary

#### Full review (magi or self-review)

Use when the fast path does NOT apply: tests not defined, tests failed, inner loop returned `partial` or `blocked`, or `tests_status: not_run`.

```
/magi "Review this implementation work:

## Work Summary
[inner loop's returned summary]

## Work Unit
[what was being implemented]

## Evaluate
1. Implementation correctness - Does it work? Tests pass?
2. Alignment - Did the work match the intended unit?
3. Gap discovery - Any new gaps, edge cases, or TODOs?
4. Completeness - Is the overall work fully realized?

Return structured assessment:
- verdict: pass | fail | needs_work
- gaps_discovered: [list]
- remaining_work: [description]
- recommendation: continue | needs_human_input | archive
- rationale: [brief explanation]"
```

**Fallback (self-review)**: If magi unavailable, review the work yourself using these criteria:
1. **Tests pass?** Run the test suite and verify green
2. **Files changed match intent?** Compare changed files to work unit scope
3. **New gaps?** Grep for TODO, FIXME, or incomplete implementations
4. **Edge cases?** Check error handling and boundary conditions
5. **Verdict**: Apply same pass/fail/needs_work logic as magi would

### 5. UPDATE STATE

Update the state file per guidance:
- **Default**: Update `progress` array — set section status, append gaps/edge cases to `notes`
- **Per guidance**: Check off criteria, append to sections, etc.

**Update guardrail tracking:**
- Increment `iterations`
- If inner loop returned `status: partial` AND `files_changed` is empty:
  - Increment `no_progress_count`
- Else:
  - Reset `no_progress_count` to 0

Set review timestamp (`last_review`).

**Auto-commit at iteration boundary:**
After updating the state file, commit all changes from this iteration:
```bash
git add -A && git commit -m "ralph: iteration [N] - [work unit name]"
```
This provides cheap rollback per iteration via `git revert` or `git reset` without needing per-run directories.

### 6. ROUTE

Based on review recommendation:

**`continue`**:
- Add any `gaps_discovered` from review to the current work unit's `notes` in progress
- Go to step 2

Note: This naturally handles "conditional pass" - issues are tracked and prevent archiving until resolved.

**`needs_human_input`**:
- Use AskUserQuestion to surface the decision
- Present: what was attempted, what needs clarification, options
- After response → step 2

**`archive`** (or equivalent completion):
- Verify no remaining gaps/issues per guidance
- Mark status as `archived` in frontmatter
- Move file to `plans/archived/` (or per guidance)
- Stage only relevant files: state file + files from `files_changed` arrays
- Use AskUserQuestion to confirm commit: "Ready to commit completion of [plan title]. Proceed?"
- If confirmed: `git commit -m "Complete: [plan title]"`
- If issues remain → inform user, continue to step 2

## Completion Criteria

The loop completes when:
1. Review assesses work as "fully realized"
2. No remaining gaps or blockers
3. Per-guidance completion signals sent (if applicable)

## Error Handling

| Scenario | Action |
|----------|--------|
| Magi unavailable | Self-review using fallback criteria (see step 4) |
| Inner loop blocked | Surface via AskUserQuestion with context |
| State file parse error | Show error, ask user to fix format |
| No .ralph.md | Offer to create or use defaults (see Project Guidance) |

## Manual Control

Interrupt anytime. State file preserves progress. Resume with `/ralph [state-file]`.

## Reference

- [inner-prompt.md](${CLAUDE_SKILL_DIR}/inner-prompt.md) - Inner loop subagent template
- [examples/](${CLAUDE_SKILL_DIR}/examples/) - Example .ralph.md files for different project types
- [examples/README.md](${CLAUDE_SKILL_DIR}/examples/README.md) - Guide for crafting .ralph.md files
- [docs/ARCHITECTURE.md](${CLAUDE_SKILL_DIR}/docs/ARCHITECTURE.md) - Full architecture documentation
