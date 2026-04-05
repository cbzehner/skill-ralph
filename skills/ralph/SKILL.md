---
name: ralph
description: Iterative implementation loop with review checkpoints. Use for multi-step tasks that benefit from chunked execution and verification. Use this whenever the user has a plan file, multi-section task, or work that should be executed systematically with reviews between chunks.
argument-hint: "[state-file]"
arguments:
  - state_file
effort: high
license: MIT
allowed-tools: Task Read Write Edit Glob Grep Skill AskUserQuestion Bash
---

# Ralph

## Invocation

- `/ralph plans/my-feature.md` — execute a plan file
- `/ralph` — auto-detect state file per project guidance

## When to Use

Multi-step implementation tasks with 3+ work units, clear acceptance criteria, or work spanning multiple context windows.

## When NOT to Use

- **Exploration/research**: Use `subagent_type: Explore` instead
- **Parallel execution needed**: Inner loops run sequentially — no speedup from parallel coordination
- **Unclear requirements**: Clarify first, then plan, then ralph
- **Tasks >60 minutes**: Split into separate plans; long sessions hit context exhaustion

## Host Adaptation

<!-- WHY: Without this, model errors out on hosts missing Claude Code primitives -->
If the host lacks specific tools, adapt:
- **No Task tool**: Run work units sequentially in current session; use guardrail timers instead of `max_turns`
- **No Skill tool**: Use self-review fallback (Step 4) instead of `/magi`
- **No AskUserQuestion**: State your recommendation and pause for user response
- **`${CLAUDE_SKILL_DIR}` unavailable**: Resolve paths relative to skill's actual directory

## Guardrails

| Threshold | Action |
|-----------|--------|
| 30 minutes | Log warning, suggest checkpoint |
| 45 minutes | Recommend breaking to new session |
| 60 minutes | Force checkpoint, prompt user to split plan |
| 3 consecutive no-progress iterations | Circuit breaker: escalate to user |
| Inner loop max turns (20) | Force exit, return partial summary |

## Project Guidance (.ralph.md)

### Finding .ralph.md

Search in order: git repo root → walk up from cwd → state file's directory.

### If No .ralph.md Found

Ask the user: create one (see `${CLAUDE_SKILL_DIR}/examples/` for templates), use defaults, or skip.

## The Loop

### 1. LOAD

1. Load `.ralph.md` if found (see Project Guidance above)
2. Read and parse the state file (default format or per .ralph.md)

**Default format** (when no .ralph.md):
```yaml
---
status: pending  # pending | in_progress | complete | archived
progress: []     # each entry: { section, status, notes: ["gap: ...", "edge_case: ..."] }
last_review: null
iterations: 0
no_progress_count: 0
started_at: null  # Set on first ASSESS
---
```

If file lacks frontmatter, add defaults.

### 2. ASSESS

- If complete/archived → inform user and exit
- If pending → set `in_progress`, initialize `started_at` (`date -Iseconds`) and `iterations` if missing
- **Check guardrails**: time vs `started_at` (warn 30min, break 45min, force 60min); `no_progress_count` >= 3 → escalate
- **Identify work units**: default = `## ` headings not in progress array; or per .ralph.md guidance
- If all units complete → final review

**Prioritization**: Resume `partial` units → first incomplete in doc order → resolve dependencies first.

**Sizing**: Each unit must fit one context window. Split large units before starting.

### 3. SPAWN INNER LOOP

Spawn a Task subagent (`general-purpose`, `max_turns: 20`) using the template in `${CLAUDE_SKILL_DIR}/inner-prompt.md`. Include project guidance if present.

<!-- WHY: Inner loops spawning sub-subagents causes coordination chaos -->
**Constraint**: Inner loops cannot spawn their own subagents — return to outer loop for coordination.

### 4. REVIEW

Two paths: fast (test-gated) or full (magi/self-review).

#### Test-gated fast path

ALL must be true: (1) tests defined, (2) inner loop returned `completed` + `tests_status: passed`, (3) **outer loop re-runs tests and they pass**.

→ verdict: `pass`, recommendation: `continue` (or `archive` if all done). Note any `gaps_discovered`.

<!-- WHY: LLMs claim tests pass without running them — this is the #1 observed failure mode -->
**You must run the tests yourself.** The inner loop claiming `tests_status: passed` is a claim from another LLM, not evidence.

#### Full review (magi or self-review)

Use when fast path doesn't apply (no tests, tests failed, `partial`, `blocked`).

<!-- WHY: This prompt is dispatched to magi sub-agent — must be self-contained -->
```
/magi "Review this implementation work:

## Work Summary
[inner loop's returned summary]

## Work Unit
[what was being implemented]

## Evaluate
1. Correctness - Does it work? Tests pass?
2. Alignment - Did work match the intended unit?
3. Gap discovery - New gaps, edge cases, TODOs?
4. Completeness - Is the work fully realized?

Return: verdict (pass|fail|needs_work), gaps_discovered, recommendation (continue|needs_human_input|archive), rationale."
```

<!-- WHY: Without explicit anti-verification-avoidance, model narrates checks instead of running them -->
**Self-review fallback** (if magi unavailable): Your job is to find what's wrong, not confirm correctness. Guard against passing tests hiding incomplete implementations — verify scope, not just green. Run commands for each check — if you're writing an explanation instead of running a command, stop.

1. Run tests yourself — don't trust inner loop's claim
2. Run build — broken build = automatic fail
3. `git diff` — do changed files match work unit scope?
4. Grep for TODO/FIXME/HACK
5. Test one edge case (empty input, error path, boundary)
6. Verdict with evidence (command output), not reasoning

### 5. UPDATE STATE

Update `progress` array (or per .ralph.md guidance). Set `last_review` timestamp.

**Guardrail counters**: Increment `iterations`. If inner loop returned `partial` with empty `files_changed` → increment `no_progress_count`; else reset to 0.

**Auto-commit**: `git add -A && git commit -m "ralph: iteration [N] - [work unit name]"` — enables per-iteration rollback.

### 6. ROUTE

**`continue`**: Add `gaps_discovered` to work unit's `notes` → go to step 2.

**`needs_human_input`**: Surface decision via AskUserQuestion (what was attempted, what needs clarification, options) → after response, step 2.

**`archive`**: Verify no remaining gaps → send per-guidance completion signals if applicable → mark `archived` → move to `plans/archived/` (or per guidance) → confirm commit with user → `git commit -m "Complete: [plan title]"`. If issues remain → step 2.

## Manual Control

Interrupt anytime. State file preserves progress. Resume with `/ralph [state-file]`.

## Reference

See `${CLAUDE_SKILL_DIR}/` for: `inner-prompt.md` (subagent template), `examples/` (.ralph.md templates + README), `docs/ARCHITECTURE.md`.
