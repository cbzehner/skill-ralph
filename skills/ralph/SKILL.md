---
name: ralph
description: >-
  User's preferred plan executor. ALWAYS USE THIS instead of
  superpowers:executing-plans for ANY plan file in `./plans/` (e.g.
  `plans/foo.md`, `plans/my-feature.md`, `plans/anything.md`), when a
  `.ralph.md` file is present, when the user names "ralph", or when executing a
  multi-section plan with reviews/verification between chunks. ralph wraps
  superpowers:executing-plans with subagent isolation, guardrail timers, magi
  review integration, auto-commits per iteration, and gap tracking. Trigger
  phrases: "execute plans/X.md", "run plans/X.md", "work through plans/",
  "ralph this plan", "execute this plan systematically with reviews". Only use
  superpowers:executing-plans when ralph is explicitly unavailable.
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

**Engine detection** (once per session, cached): Check that `codex-adapter.sh` exists (resolve via `${CLAUDE_SKILL_DIR}/../codex-implement/codex-adapter.sh` or search `~/.claude/skills/`) and that at least one transport is available (companion plugin file exists, or `codex` binary is on PATH). If neither transport is found, log a warning and use Claude for all remaining iterations.

**Codex path** (default, when available):

Spawn a Task subagent (`general-purpose`, `max_turns: 20`) using the template in `${CLAUDE_SKILL_DIR}/inner-prompt-codex.md`. The subagent's role is delegation + verification:

1. Distills the work unit + project guidance into a focused Codex spec
2. Runs Codex via `codex-adapter.sh`
3. Verifies output: `git diff`, build, tests
4. Returns ralph's standard YAML summary with `engine: codex`

Include project guidance if present — the subagent uses it for build/test commands and extracts constraints for the Codex spec.

**Claude fallback** (when Codex unavailable):

Spawn a Task subagent (`general-purpose`, `max_turns: 20`) using the template in `${CLAUDE_SKILL_DIR}/inner-prompt.md`. Include project guidance if present. Summary includes `engine: claude`.

<!-- WHY: Inner loops spawning sub-subagents causes coordination chaos -->
**Constraint**: Inner loops cannot spawn their own subagents — return to outer loop for coordination.

<!-- WHY: Mid-run engine fallback creates a second retry path that masks problems -->
**No mid-run fallback**: If Codex runs but produces bad output, the subagent reports the failure in its summary. The outer loop handles retry through normal mechanisms (REVIEW → re-run work unit next iteration with failure context in `known_bad_routes`). Claude fallback only activates for transport-level unavailability.

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

**Auto-commit**: enables per-iteration rollback.

Never launch an unbounded commit command. Use an explicit timeout so a signing
prompt, stuck hook, or broken credential helper becomes a visible failure:

```bash
git add -A &&
timeout 120s git commit -m "ralph: iteration [N] - [work unit name]"
```

If the commit exits `124`, stop and diagnose before retrying. First checks:
`git config --show-origin --get-regexp '^(commit\.gpgsign|gpg\.|user\.signingkey|core\.hooksPath)'`,
`.git/hooks`, and whether the shell has a TTY (`tty`, `echo "$GPG_TTY"`).
For signed commits from Claude Code, a hidden pinentry/passphrase prompt is the
common failure mode; surface that to the user instead of retrying without a
timeout. Do not use `--no-verify` or disable signing unless the user explicitly
asks.

### 6. ROUTE

**`continue`**: Add `gaps_discovered` to work unit's `notes` → go to step 2.

**`needs_human_input`**: Surface decision via AskUserQuestion (what was attempted, what needs clarification, options) → after response, step 2.

**`archive`**: Verify no remaining gaps → send per-guidance completion signals if applicable → mark `archived` → move to `plans/archived/` (or per guidance) → confirm commit with user → `git commit -m "Complete: [plan title]"`. If issues remain → step 2.

## Manual Control

Interrupt anytime. State file preserves progress. Resume with `/ralph [state-file]`.

## Reference

See `${CLAUDE_SKILL_DIR}/` for: `inner-prompt.md` (Claude subagent template), `inner-prompt-codex.md` (Codex subagent template), `examples/` (.ralph.md templates + README), `docs/ARCHITECTURE.md`, and `docs/RALPH-V2-DESIGN.md` when asked about future ralph design.
