# .ralph.md - Task-Based Project (Launchpad-style)

For projects using task files with acceptance criteria, like launchpad.

## State Files

Tasks live in `.launchpad/tasks/` with this structure:

```yaml
---
id: task-001
status: queued | planning | implementing | complete | stuck
labels: []
verification_attempts: 0
---

# Task Title

## Objective
What to accomplish (1-3 sentences).

## Acceptance Criteria
- [ ] First criterion
- [ ] Second criterion
- [ ] Third criterion

## Planning
Agent fills this before implementation.

## Known Bad Routes
Approaches that failed - avoid repeating these.

## Verified Findings
Validated facts about the codebase.

## Magi Feedback
Verification history from /done --verify calls.
```

## Work Units

Each unchecked acceptance criterion (`- [ ]`) is a work unit.

**Grouping**: Group related criteria that share implementation context.
For example, if criteria 1-2 both involve the same component, work them together.

**Order**: Work criteria in logical dependency order, not necessarily document order.

## Progress Tracking

- Check off criteria (`- [x]`) as they're completed
- Append to `## Known Bad Routes` when an approach fails
- Append to `## Verified Findings` when facts are validated
- `## Magi Feedback` is updated automatically by `/done --verify`

## Review

**After each work unit**: Self-review (verify tests pass, code compiles).

**Before signaling done**: Use `/done --verify` for magi verification.
Magi checks implementation against acceptance criteria.

**If verification fails**: Address feedback, try again. After 5 failed attempts,
the task signals stuck automatically.

## Completion

Environment variables (set by launchpad coordinator):
- `LAUNCHPAD_TASK_ID` - Current task identifier
- `LAUNCHPAD_TASK_FILE` - Path to task file

**When all criteria met and verified**: `/done`
**When blocked and need human help**: `/stuck "reason"`

The coordinator processes these signals and routes appropriately.

## Context

This project uses launchpad for multi-agent orchestration.
The coordinator is external - it dispatches tasks and processes signals.
Your job is to complete the work and signal appropriately.

Read the full task file at start to understand:
- Objective and acceptance criteria
- Known bad routes to avoid
- Verified findings to build on
- Prior magi feedback if any
