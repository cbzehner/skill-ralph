# Crafting .ralph.md Files

A `.ralph.md` file provides project-specific guidance for the ralph iterative implementation loop. It tells Claude how your project organizes work, tracks progress, and signals completion.

## Purpose

Without a `.ralph.md`, ralph uses default plan-file conventions:
- State in YAML frontmatter + markdown sections
- Work units = `## ` headings
- Progress = frontmatter arrays
- Review = magi synthesis

A `.ralph.md` lets you adapt this to your project's conventions.

## Location

Place `.ralph.md` at your **git repository root**. Ralph finds it via:
1. `git rev-parse --show-toplevel` → check for `.ralph.md`
2. Walk up directory tree as fallback

## Structure

A `.ralph.md` is **markdown that Claude reads as guidance**, not a config file to parse. Write it as instructions Claude can follow.

### Recommended Sections

```markdown
# .ralph.md - Project Name

## State Files
Where state lives and how it's structured.

## Work Units
How to identify what to work on (sections, criteria, issues).

## Progress Tracking
How to record progress (frontmatter, checkboxes, comments).

## Review
When and how to review work (magi, human, CI, tests).

## Completion
How to signal done (archive, /done skill, PR creation).

## Context (Optional)
Project-specific knowledge for inner loops.
```

## Design Principles

### 1. Be Declarative, Not Procedural

**Good**: "Work units are unchecked acceptance criteria in the task file"
**Bad**: "Parse the file, find lines starting with '- [ ]', filter out checked ones..."

Claude understands natural language. Describe *what*, not *how*.

### 2. Explain the Why

Help Claude make good decisions by explaining rationale:

```markdown
## Review

Use magi synthesis after each work unit because this project has
complex acceptance criteria that benefit from multi-model verification.

Skip magi for documentation-only changes (low risk, tests sufficient).
```

### 3. Reference Your File Formats

If your state files have a specific structure, document it:

```markdown
## State Files

Task files at `.launchpad/tasks/{id}.md` with structure:

- YAML frontmatter: id, status, labels
- `## Objective`: What to accomplish
- `## Acceptance Criteria`: Checkboxes (work units)
- `## Known Bad Routes`: Failed approaches to avoid
```

### 4. Define Exit Signals

If your project has orchestration (like launchpad), specify how to signal:

```markdown
## Completion

When all criteria met: `/done` (or `/done --verify` for magi review)
When blocked: `/stuck "reason"`

Environment variables available:
- LAUNCHPAD_TASK_ID
- LAUNCHPAD_TASK_FILE
```

### 5. Keep It Focused

A `.ralph.md` should be ~50-150 lines. If longer, you're over-specifying.

Don't repeat what's in the skill. Focus on project-specific adaptations.

## Examples in This Directory

| File | Use Case |
|------|----------|
| `plan-based.ralph.md` | Traditional plan files with ## sections |
| `task-based.ralph.md` | Task files with acceptance criteria |
| `minimal.ralph.md` | Simplest possible guidance |
| `github-issues.ralph.md` | GitHub issues as work units |

## Common Patterns

### Task-Based Projects (Launchpad-style)

```markdown
## Work Units
Each unchecked acceptance criterion is a work unit.
Group related criteria that share implementation context.
Check off criteria as completed.

## Completion
/done --verify  # With magi review
/stuck "reason" # When blocked
```

### Plan-Based Projects (Original ralph)

```markdown
## Work Units
Each ## heading is a section to implement.
Work sections in order unless dependencies require otherwise.

## Progress Tracking
Add completed section names to `progress:` array in frontmatter.
```

### CI-Integrated Projects

```markdown
## Review
After each work unit: run `npm test` and verify passing.
Before completion: run full CI suite locally.
Skip magi - CI provides sufficient verification.
```

### Human-Review Projects

```markdown
## Review
After implementation: commit changes and create draft PR.
Signal needs_human_input for review.
Resume after human approves or requests changes.
```

## Anti-Patterns

### Over-Specification

**Bad**: Specifying every edge case and error handling path
**Good**: Trust Claude to handle standard situations; document only what's unique

### Procedural Instructions

**Bad**: "First read file X, then parse line Y, then check condition Z..."
**Good**: "State is in X with format Y. Work units are Z."

### Duplicating the Skill

**Bad**: Re-explaining the core loop mechanics
**Good**: Only documenting project-specific adaptations

### Rigid Formatting

**Bad**: "Output must be exactly: `STATUS: [status]\nGAPS: [gaps]`"
**Good**: Let Claude use natural formatting; specify only when integration requires it

## Testing Your .ralph.md

1. Run `/ralph` on a small task
2. Check that Claude correctly identifies work units
3. Verify progress tracking works as expected
4. Confirm completion signals are correct

If Claude misinterprets something, clarify that section of your `.ralph.md`.

## Generating .ralph.md

If you don't have a `.ralph.md` and run `/ralph`, you'll be offered the option to create one. The skill will ask questions about:

- Where your state files live
- How work is organized (sections, criteria, issues)
- What review process you want
- How to signal completion

Based on your answers, it generates a starter `.ralph.md` you can refine.
