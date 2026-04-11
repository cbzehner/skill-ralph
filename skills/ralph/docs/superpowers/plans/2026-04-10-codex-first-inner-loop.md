# Codex-First Inner Loop Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make ralph's inner loop use Codex by default, falling back to Claude when Codex is unavailable.

**Architecture:** Step 3 (SPAWN INNER LOOP) gains engine detection and a Codex delegation path. A new `inner-prompt-codex.md` template instructs the Task subagent to delegate to Codex via `codex-adapter.sh` and verify the results. The outer loop is unchanged.

**Tech Stack:** Bash (codex-adapter.sh), YAML (ralph state), Markdown (skill files)

**Spec:** `docs/superpowers/specs/2026-04-10-codex-first-inner-loop-design.md`

---

### Task 1: Create `inner-prompt-codex.md`

New subagent template for the Codex delegation path. The subagent's role is delegation + verification, not direct implementation.

**Files:**
- Create: `inner-prompt-codex.md` (in ralph skill dir)
- Reference: `inner-prompt.md` (existing Claude template, for summary format consistency)
- Reference: `codex-implement/codex-adapter.sh` (transport layer)
- Reference: `codex-implement/references/subagent-prompt.md` (Codex subagent patterns)

- [ ] **Step 1: Create `inner-prompt-codex.md`**

Write the template. Key differences from `inner-prompt.md`:
- The subagent does NOT implement directly — it builds a spec, runs Codex, and verifies
- It uses `codex-adapter.sh` for transport
- It captures a git baseline before Codex runs and diffs after
- It runs build/tests to verify Codex's output
- It returns the same YAML summary format as the Claude path, plus `engine: codex`

```markdown
# Inner Loop Subagent Prompt Template (Codex Engine)

Use this template when spawning the inner loop Task subagent with Codex as the implementation engine.
The subagent's role is delegation + verification — it does not implement directly.

## Template

~~~markdown
You are delegating implementation work to OpenAI Codex and verifying the results.
Work through the steps below, then summarize and exit.

{{#if PROJECT_GUIDANCE}}
## Project Guidance

{{PROJECT_GUIDANCE}}
{{/if}}

## Current Focus

**Work Unit:** {{WORK_UNIT_NAME}}

**Goal:** {{WORK_UNIT_CONTENT}}

## Instructions

### 1. Build the Codex Spec

Write a focused implementation spec to a temp file. Include:

- **Goal**: What to implement (from Work Unit above)
- **Target files**: Which files to create or modify
- **Constraints**: Any hard rules from Project Guidance
{{#if KNOWN_BAD_ROUTES}}
- **Avoid these approaches** (they failed previously):
{{KNOWN_BAD_ROUTES}}
{{/if}}
{{#if VERIFIED_FINDINGS}}
- **Established facts** (build on these):
{{VERIFIED_FINDINGS}}
{{/if}}

Keep the spec focused. Include only what Codex needs to implement this one work unit.
Do NOT paste the full state file — distill it into actionable instructions.

Write the spec:
```bash
spec_dir=$(mktemp -d)
cat > "$spec_dir/spec.md" << 'SPEC'
[your spec content here]
SPEC
```

### 2. Capture Baseline

```bash
base_sha=$(git rev-parse HEAD)
```

### 3. Run Codex

```bash
bash {{CODEX_ADAPTER_PATH}} "$(cat $spec_dir/spec.md)"
```

If Codex exits non-zero, capture the error and skip to the summary with `status: blocked`.

### 4. Verify Codex Output

Your primary job — Claude verifying Codex's work:

1. `git diff "$base_sha"` — review all changes, check they match the intended scope
2. Run the project's build command — broken build is noteworthy
3. Run the project's test command — failed tests are noteworthy
4. Check for unexpected file changes outside the work unit scope

If you're writing an explanation instead of running a command, stop and run the command.

### 5. Return Summary

**Do NOT spawn subagents** — return to outer loop for coordination.
**Do NOT commit** — the outer loop handles commits at iteration boundaries.

```yaml
status: completed | partial | blocked
work_done:
  - "Description of what Codex accomplished (from git diff)"
blockers:
  - "Any Codex errors or verification failures (empty if none)"
gaps_discovered:
  - "Unexpected changes, missing coverage, new edge cases"
files_changed:
  - path/to/file.ext
tests_status: passed | failed | not_run
next_steps:
  - "Follow-up items from verification"
engine: codex
```

## Exit Criteria

Stop and return your summary when ANY of these apply:
- Codex completed + verification passed → `status: completed`
- Codex failed or verification found issues → `status: partial` (with details)
- Codex transport error → `status: blocked`
- Turn limit → max 20 turns reached → `status: partial`
~~~

## Substitution Variables

Required:
- `{{WORK_UNIT_NAME}}`: The work unit being implemented (section heading, criterion text, etc.)
- `{{WORK_UNIT_CONTENT}}`: Details about the work unit (section content, related context)
- `{{CODEX_ADAPTER_PATH}}`: Absolute path to `codex-adapter.sh` (resolved at runtime)

Optional (include if present in state file):
- `{{PROJECT_GUIDANCE}}`: Contents of .ralph.md if found
- `{{KNOWN_BAD_ROUTES}}`: Approaches that failed (from state file)
- `{{VERIFIED_FINDINGS}}`: Validated facts (from state file)

## Notes

The template uses mustache-style conditionals (`{{#if}}...{{/if}}`) for optional sections.
If a variable is not available, omit that section entirely.

Unlike the Claude inner prompt, this template does NOT include `{{FULL_STATE_MARKDOWN}}`.
Codex works best with focused specs — the subagent distills relevant context into the spec
rather than forwarding the entire state file.

Project guidance is included for the subagent (so it knows build/test commands and conventions),
but is NOT forwarded verbatim to Codex — the subagent extracts relevant constraints into the spec.
```

- [ ] **Step 2: Verify template consistency**

Check that the YAML summary format matches `inner-prompt.md` exactly (same fields, same status values), plus the new `engine` field:

```bash
# Extract summary format from both templates and compare
grep -A 20 "^status:" inner-prompt.md
grep -A 20 "^status:" inner-prompt-codex.md
```

- [ ] **Step 3: Commit**

```bash
git add inner-prompt-codex.md
git commit -m "ralph: add Codex inner loop subagent template"
```

---

### Task 2: Update SKILL.md Step 3

Modify step 3 (SPAWN INNER LOOP) to describe Codex-first behavior with Claude fallback.

**Files:**
- Modify: `SKILL.md:92-97` (step 3 section)
- Modify: `SKILL.md:167` (reference section — add inner-prompt-codex.md)

- [ ] **Step 1: Replace step 3 content**

Replace lines 92-97 of `SKILL.md` with the Codex-first inner loop description:

````markdown
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
````

- [ ] **Step 2: Update the Reference section**

Replace the last line of SKILL.md (the Reference section) to include the new template:

```markdown
See `${CLAUDE_SKILL_DIR}/` for: `inner-prompt.md` (Claude subagent template), `inner-prompt-codex.md` (Codex subagent template), `examples/` (.ralph.md templates + README), `docs/ARCHITECTURE.md`.
```

- [ ] **Step 3: Verify no broken internal references**

```bash
# Check all ${CLAUDE_SKILL_DIR} references resolve to real files
grep -o '\${CLAUDE_SKILL_DIR}/[^ ]*' SKILL.md | sed 's/`//g' | sort -u
```

Confirm each path maps to a file that exists (or will exist after Task 1).

- [ ] **Step 4: Commit**

```bash
git add SKILL.md
git commit -m "ralph: update step 3 for Codex-first inner loop"
```

---

### Task 3: Update ARCHITECTURE.md

Update the architecture doc to reflect the dual-engine inner loop.

**Files:**
- Modify: `docs/ARCHITECTURE.md:1-176` (diagram and inner loop sections)

- [ ] **Step 1: Update the overview diagram**

Replace the ASCII diagram (lines 12-39) with one showing the dual-engine inner loop:

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

- [ ] **Step 2: Update the "Why Nested Loops?" section**

Add a paragraph after the existing content (after line 47):

```markdown
**Why Codex First?**: Codex is faster for scoped implementation tasks. Ralph already sizes work units to fit one context window with clear goals — a natural match. Using Codex for implementation while Claude handles orchestration and review gives the best of both engines. When Codex isn't available, the original Claude inner loop remains as a fallback.
```

- [ ] **Step 3: Add "Engine Detection" section**

Add a new section after "Inner Loop Scope" (after line 59):

```markdown
## Engine Detection

Ralph detects Codex availability once per session and caches the result:

1. Locate `codex-adapter.sh` (from codex-implement skill directory)
2. Check that at least one transport is available:
   - Companion plugin file exists (`codex-companion.mjs`), OR
   - `codex` binary is on PATH

If neither transport is found, ralph logs a warning and uses the Claude inner loop for all iterations. This is a filesystem check, not a live Codex invocation.

**No mid-run fallback**: If Codex runs but produces bad output, the subagent reports it and the outer loop handles retry. Claude fallback only activates for transport-level unavailability detected at session start.
```

- [ ] **Step 4: Update "Inner Loop Subagent" section**

Replace the inner loop subagent section (lines 172-175) to describe both engines:

```markdown
## Inner Loop Subagent

The inner loop is a Task subagent with limited tool access. Inner loops **cannot spawn their own subagents** and **do not commit** — the outer loop handles both coordination and commits at iteration boundaries.

**Codex engine** (default): The subagent delegates to Codex via `codex-adapter.sh` and verifies the results (git diff, build, tests). See `inner-prompt-codex.md` for the prompt template.

**Claude engine** (fallback): The subagent implements directly using Read, Write, Edit, Bash, Glob, Grep. See `inner-prompt.md` for the prompt template.

Both engines return the same YAML summary format. The `engine` field indicates which ran.
```

- [ ] **Step 5: Commit**

```bash
git add docs/ARCHITECTURE.md
git commit -m "ralph: update architecture docs for Codex-first inner loop"
```
