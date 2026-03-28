# .ralph.md - GitHub Issues Integration

For projects that track work as GitHub issues.

## State Files

Work is tracked in GitHub issues, not local files.

When `/ralph` is invoked with an issue number:
```
/ralph #123
```

Fetch the issue:
```bash
gh issue view 123 --json title,body,labels,state
```

## Work Units

The issue body contains the work specification.

If the issue has a checklist (task list), each unchecked item is a work unit:
```markdown
- [ ] Implement feature X
- [ ] Add tests for X
- [ ] Update documentation
```

If no checklist, treat the entire issue as a single work unit.

## Progress Tracking

Update the issue as work progresses:

```bash
# Add a comment with progress
gh issue comment 123 --body "Completed: Implement feature X

Files changed:
- src/feature.ts
- tests/feature.test.ts"

# Check off completed items (edit issue body)
# Or add labels like "in-progress", "needs-review"
```

## Review

After each work unit:
1. Run tests locally
2. Add progress comment to issue
3. Use magi for complex implementations

Before completion:
1. Create PR linked to issue
2. Request review if required by project

## Completion

When all work units complete:

```bash
# Create PR
gh pr create --title "Fix #123: [issue title]" --body "Closes #123"

# Or if no PR needed, close the issue
gh issue close 123 --comment "Completed in commit [sha]"
```

## Context

Use `gh` CLI for all GitHub operations. It's already authenticated.

Common commands:
```bash
gh issue view 123              # View issue
gh issue comment 123 --body "" # Add comment
gh issue edit 123 --body ""    # Update issue body
gh pr create                   # Create pull request
gh issue close 123             # Close issue
```
