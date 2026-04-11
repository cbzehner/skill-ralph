# Codex-First Inner Loop for Ralph

## Summary

Ralph's inner loop (step 3: SPAWN INNER LOOP) uses Codex by default for implementation work, falling back to the current Claude Task subagent only when Codex is unavailable at the transport level. The outer loop (LOAD, ASSESS, REVIEW, UPDATE, ROUTE) is unchanged.

## Motivation

Codex is faster and cheaper for scoped implementation tasks. Ralph already sizes work units to fit one context window with clear goals — a natural match for Codex's strengths. Using Codex for implementation while Claude handles orchestration and review gives us the best of both engines.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     OUTER LOOP                          │
│              (unchanged — your Claude session)          │
│                                                         │
│  LOAD → ASSESS → SPAWN INNER LOOP → REVIEW → UPDATE → ROUTE
│                        │                                │
│                        ▼                                │
│              ┌─── Try Codex ───┐                       │
│              │                  │                       │
│              ▼                  ▼                       │
│        ┌──────────┐    ┌──────────────┐                │
│        │  Codex   │    │ Claude Task  │                │
│        │ (default)│    │ (fallback)   │                │
│        └──────────┘    └──────────────┘                │
│              │                  │                       │
│              └──── returns ─────┘                       │
│                    ralph YAML                           │
│                    summary                              │
└─────────────────────────────────────────────────────────┘
```

## Design

### Engine Detection

Runs once per ralph session, result cached for remaining iterations.

Detection method: check that `codex-adapter.sh` exists and that at least one transport is available (companion plugin file exists, or `codex` binary is on PATH). This is a filesystem check, not a live invocation — avoids burning Codex credits on a probe. If neither transport is found, log a warning and use Claude for all remaining iterations.

The adapter (`codex-implement/codex-adapter.sh`) already handles transport selection (companion plugin vs CLI fallback), so ralph delegates that concern entirely.

### Codex Path (Default)

When Codex is available, the outer loop spawns a Task subagent (`general-purpose`) whose role is **delegation + verification**, not direct implementation.

The subagent follows a new template (`inner-prompt-codex.md`) with these steps:

#### 1. Build Spec

Distill the work unit into a focused Codex prompt. Include:

- **Goal**: Work unit name and content
- **File scope**: Target files to modify, files not to touch
- **Constraints**: Hard constraints from project guidance (`.ralph.md`)
- **Avoid**: Known bad routes, formatted as explicit "do NOT" instructions
- **Build on**: Verified findings, formatted as established facts

Write the spec to a temp file (`tmp/ralph-codex-spec.md`). Codex works best with clear, scoped instructions — avoid dumping the full state file into the prompt.

#### 2. Capture Baseline

```bash
git rev-parse HEAD
```

Record the SHA before Codex runs, so we can diff precisely.

#### 3. Run Codex

```bash
bash /path/to/codex-adapter.sh "$(cat tmp/ralph-codex-spec.md)"
```

The adapter prefers the companion plugin (HTTP) and falls back to `codex exec` CLI with a 300s timeout.

#### 4. Verify

The subagent's primary value — Claude verifying Codex's work:

- `git diff $base_sha` — review all changes
- Run build command — broken build is noteworthy
- Run tests — failed tests are noteworthy
- Check that changed files match the intended scope

#### 5. Return Ralph YAML Summary

Translate results into ralph's standard format:

```yaml
status: completed | partial | blocked
work_done:
  - "Description from git diff summary"
blockers:
  - "Any Codex errors or verification failures"
gaps_discovered:
  - "Unexpected changes, missing coverage, new edge cases"
files_changed:
  - path/to/file.ext
tests_status: passed | failed | not_run
next_steps:
  - "Follow-up items from verification"
engine: codex
```

The `engine` field is new — lets the outer loop log which engine executed. Both paths produce the same summary shape.

### Claude Fallback Path

Activated only when Codex is unavailable at the transport level (detection failed). Uses the existing `inner-prompt.md` template unchanged. Adds `engine: claude` to the summary.

### No Mid-Run Fallback

If Codex runs but produces bad output (wrong files, build fails, tests fail), the subagent does NOT retry with Claude. It reports the failure in the summary and the outer loop handles it through normal mechanisms:

- REVIEW step catches the failure (test-gated fast path or magi)
- Outer loop re-runs the work unit next iteration
- Failure context flows into `known_bad_routes` for the next attempt
- If 3 consecutive no-progress iterations occur, circuit breaker escalates to user

Rationale: adding a mid-run engine fallback creates a second retry path that masks problems and complicates debugging. Transport errors vs implementation errors deserve different handling.

### Error Handling

| Error | When | Action |
|-------|------|--------|
| Codex binary missing | Detection | Fall back to Claude for session |
| Auth/permission failure | Detection | Fall back to Claude for session |
| Codex timeout (>300s) | During run | Subagent returns `status: partial`, outer loop retries |
| Codex produces wrong output | After run | Subagent returns with verification failures, outer loop retries |
| `codex-adapter.sh` not found | Detection | Fall back to Claude, log warning about missing codex-implement skill |

## What Changes

### Files to Modify

1. **`ralph/SKILL.md`** — Update step 3 to describe Codex-first with Claude fallback behavior
2. **`ralph/docs/ARCHITECTURE.md`** — Updated diagram showing dual-engine inner loop

### Files to Create

3. **`ralph/inner-prompt-codex.md`** — New subagent template for the Codex delegation path

### Dependencies

- `codex-implement/codex-adapter.sh` — ralph references this script directly for Codex transport. No changes needed to that file.

## What Doesn't Change

- **LOAD** — same
- **ASSESS** — same (guardrails, prioritization, sizing)
- **REVIEW** — same (test-gated fast path or magi); Claude reviewing Codex's work is a stronger signal than Claude reviewing its own work
- **UPDATE** — same, plus logs `engine` field from summary
- **ROUTE** — same
- **State file format** — same YAML frontmatter
- **`.ralph.md`** — no new config fields needed
- **`inner-prompt.md`** — kept as-is for Claude fallback path

## What's NOT in Scope

- No changes to the codex-implement skill itself
- No `.ralph.md` engine configuration
- No mid-run engine fallback
- No changes to outer loop steps (LOAD, ASSESS, REVIEW, UPDATE, ROUTE)
- No changes to the review protocol (test-gated fast path or magi)
