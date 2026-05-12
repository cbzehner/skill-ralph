# Ralph v2 Design — Patterns from gnhf

**Status**: Reviewed via `/magi` 2026-05-03. Decision matrix below reflects 3/3 advisor synthesis.

**Date**: 2026-05-03

**Source of patterns**: [kunchenguid/gnhf](https://github.com/kunchenguid/gnhf) — autonomous overnight agent runner.

**Magi session**: `~/.claude/magi/sessions/2026-05-03-ralph-v2-design-review.md`

## Problem statement

Ralph v1 works but has gaps observed during real use:

1. **No rollback on failure** — failed iterations leave partial state; user has to git-reset manually
2. **No isolation between iterations** — concurrent ralph runs in the same repo conflict
3. **Single failure mode** — `no_progress_count >= 3` is the only escalation. Treats network errors and agent self-reported failures the same
4. **Time-based caps only** — guardrails fire on wall-clock, not on goal completion
5. **YAML state-file as memory** — narrative continuity across iterations is brittle (just `notes` arrays per work unit)

## Magi review correction

The original draft over-imported gnhf's "background daemon" assumptions into ralph, which is an interactive in-session skill with review checkpoints. Several patterns were reframed after review:

- **notes.md does NOT replace YAML** — it lives alongside it. Markdown is narrative memory; YAML is control state. Removing YAML strips the outer orchestrator's programmatic progress visibility.
- **Auto-rollback is recoverable, not destructive** — plain `git reset --hard` was rejected by 2/3 advisors as destroying useful failure-state context. Snapshot to a recoverable ref before reset.
- **`--stop-when` defers an LLM call into the primary prompt** — not a separate evaluation per iteration. Cheap executable predicates (test exit, no-diff, coverage) come first.
- **Failure classification is layered** — outer wrapper handles transport/auth/HTTP via exit codes; inner loop self-reports only implementation failures.

## Decision matrix (post-review)

| Pattern | Verdict | Rationale |
|---|---|---|
| 1. Worktree isolation | **Fold into v1 as opt-in `--worktree`**; v2 default with `--in-place` escape hatch + dirty-tree preflight | Codex+Claude align: opt-in now, default-with-escape later. Gemini wants defer entirely on UX grounds (user's shell stays in parent repo) — loses on parallelism use case. |
| 2. notes.md memory | **Fold into v1, alongside YAML** | High-value for narrative LLM consumption. Do NOT remove YAML — outer orchestrator needs machine-readable state for control flow. Compaction needed to prevent context bloat. |
| 3. Tiered failure handling | **Build in v2** | Necessary for unattended loops. Detection layered: outer classifies transport (exit codes/HTTP); inner self-reports only implementation failures. Keep `no_progress_count` as orthogonal signal. |
| 4. `--stop-when` predicate | **Defer (or radically simplify)** | Per-iteration LLM call is expensive and noisy. v1: cheap executable predicates only (`--stop-on-pass`, `--stop-on-no-diff`, `--stop-after-coverage 80`). v2: inject prose goal into inner prompt with `GOAL_MET` self-signal — no second LLM call. |
| 5. Auto-rollback | **Build in v2, recoverable variant only** | Plain `git reset --hard` rejected. Snapshot to `ralph/failed/<run>/<n>` ref or stash before reset. Append failure findings to notes.md. Preflight for dirty tree before any reset. Never silently destroy uncommitted user changes. |

## v1 changes (immediate)

### 1.a Worktree opt-in

Add `--worktree` flag. When present, ralph creates a worktree at `../<repo>-ralph-<plan-slug>` on a `ralph/<slug>` branch. Inner loop runs in the worktree. Compose with existing `superpowers:using-git-worktrees`.

Preflight before creation:
- Working tree clean? (else fail with explicit message)
- Branch name available? (else suffix with `-2`)
- `.env` and submodule warnings (do NOT auto-copy — flag them and let user decide)

### 1.b notes.md alongside YAML

State file gains a sibling `notes.md` (default location: same directory as the plan file, e.g. `./plans/my-feature.notes.md`).

YAML continues to track: `status`, `progress[]`, `iterations`, `no_progress_count`, `started_at`, `last_review`. **No fields removed.**

notes.md is append-only narrative. Each iteration adds one section:

```markdown
## Iteration 7 (2026-05-03 14:23) — work unit: "login form validation"

Implemented validation. All 4 unit tests pass.
Edge case found: empty email vs whitespace-only email diverge — addressed.
Out of scope: refactor of `validateEmail()` — leaving for separate plan.
```

**Compaction**: when notes.md exceeds ~10k tokens (rough heuristic: 600 lines), prepend a "## Summary so far" section synthesizing prior iterations and truncate the verbose middle. v1 implements as a manual `/ralph compact` action; v2 may automate.

**Desync prevention**: notes.md is committed alongside iteration commits. Failed iterations' notes are appended BEFORE rollback, then the rollback preserves the notes.md commit.

### 1.c Cheap stop predicates

Add v1 flags (no extra LLM call):
- `--stop-on-pass <command>` — exit when `command` returns 0 after any iteration
- `--stop-on-no-diff` — exit if an iteration produces no commits (already-done detection)
- `--stop-after-coverage <pct>` — when paired with a coverage tool

These run after each successful iteration. They're cheap and reliable.

## v2 changes (deferred)

### 2.a Worktree default

Make `--worktree` the default in v2. `--in-place` becomes the opt-out. Migration depends on observed parallel use in v1.

### 2.b Tiered failure handling

Inner loop returns:

```yaml
verdict: failed | partial | completed
error_kind: implementation | (omitted — set by outer wrapper)
```

Outer wrapper classifies based on exit code and stderr signatures:

| Signal | error_kind | Response |
|---|---|---|
| Inner reports `verdict: failed, error_kind: implementation` | agent | Retry next iteration immediately |
| HTTP 429, 5xx, network timeout, exit 124 | hard | Exponential backoff (15s, 60s, 4m, abort) |
| HTTP 401/403, "auth expired", exit 5 | permanent | Abort with clear message |
| 3 consecutive `no_progress_count` | stall | Existing v1 circuit breaker |

### 2.c Recoverable auto-rollback

On any non-`completed` iteration:

1. **Append failure notes** to notes.md (before any tree mutation)
2. **Preflight**: check tree state, refuse if user has uncommitted unrelated changes
3. **Snapshot**: `git stash push -u -m "ralph/failed/<run>/<n>"` OR commit to `ralph/failed/<run>/<n>` ref (decide based on whether to keep failed state queryable via `git log` or hidden in stash)
4. **Reset**: `git reset --hard <pre-iteration-sha>`
5. **Record**: snapshot ref name into the YAML state file under `failed_iterations[]` for later inspection

User can recover with `git stash apply <ref>` or `git diff <ref>`.

### 2.d Inner-prompt `GOAL_MET` (replaces `--stop-when` LLM call)

Inject the prose goal into the inner agent's primary system prompt:

> Stop condition for this run: <prose goal>. When you believe this condition is satisfied at the end of an iteration, include `GOAL_MET: <one-line justification>` in your final summary.

Outer loop greps for `GOAL_MET:` in inner-loop output. No second LLM call.

## Risks acknowledged from review

1. **notes.md/repo desync**: addressed by committing notes.md atomically with each iteration; failed-iteration notes append before rollback
2. **Context-window bloat in notes.md**: addressed by compaction (manual in v1, auto in v2)
3. **Orphaned worktree accumulation**: v2 should add `ralph cleanup` to prune worktrees on completed/archived plans
4. **Worktree environment drift** (`.env`, submodules, LFS): preflight warns explicitly; never auto-copies
5. **False failure classification by inner-loop self-report**: addressed by layered classification — outer wrapper handles transport/auth, inner only handles implementation
6. **Loss of programmatic visibility if YAML removed**: addressed by keeping YAML — notes.md is alongside, not replacing

## Open questions still requiring data

- How often are ralph runs invoked with a dirty working tree? Determines whether v2 default-worktree is safe.
- How often do failed iterations contain reusable partial work vs. just noise? Determines whether recoverable-rollback's complexity is worth it over plain rollback.
- Do users `cd` into worktrees to inspect ralph progress, or do they want changes in their current checkout? UX question for the v2 default decision.

## Migration sequence

**Phase A — v1 additive (this sprint)**:
1. `--worktree` opt-in flag
2. notes.md alongside YAML (default-on once tested)
3. `--stop-on-pass`, `--stop-on-no-diff`, `--stop-after-coverage` cheap predicates
4. notes.md `/ralph compact` action

**Phase B — v2 build (after Phase A bakes 2+ weeks)**:
1. Tiered failure handling (layered classification)
2. Recoverable auto-rollback with snapshot refs
3. `GOAL_MET` inner-prompt injection (replaces `--stop-when` LLM call)
4. `ralph cleanup` for worktree pruning

**Phase C — v2 release**:
1. Worktree becomes default; `--in-place` opt-out
2. Remove deprecated `no_progress_count` only logic in favor of layered failure handling

## Non-goals

- **Replicating gnhf entirely**: gnhf is a standalone CLI for "while you sleep". Ralph is in-session. Different shapes.
- **Multi-agent orchestration**: gnhf supports concurrent agents on different worktrees. Ralph stays single-agent per run.
- **Replacing magi/self-review checkpoints**: a key ralph differentiator vs. gnhf. Stay.
