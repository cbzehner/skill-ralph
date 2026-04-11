# Inner Loop Subagent Prompt Template (Codex Engine)

Use this template when spawning the inner loop Task subagent with Codex as the implementation engine.
The subagent's role is delegation + verification — it does not implement directly.

## Template

````markdown
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

If Codex exits non-zero, capture the error output and determine the failure type:
- **Transport/auth failure** (command not found, permission denied, connection refused) → `status: blocked`
- **Implementation failure** (Codex ran but errored, timeout, bad output) → `status: partial` with error details in `blockers`

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
````

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
