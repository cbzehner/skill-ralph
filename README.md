# Ralph

Iterative implementation loops with review checkpoints. What could possibly go wrong?

> *"More Ralphs is always better, rite?"*
>
> An outer Ralph orchestrates. Inner Ralphs implement. Magi reviews.
> It's Ralphs all the way down.

```
        ┌──────────────────────────┐
        │      OUTER RALPH         │
        │    (the orchestrator)    │
        │            │             │
        │            ▼             │
        │   ┌────────────────┐     │
        │   │  INNER RALPH   │     │
        │   │  (does work)   │     │
        │   └────────────────┘     │
        │            │             │
        │            ▼             │
        │      /magi review        │
        │            │             │
        │            ▼             │
        │    repeat until done     │
        │    (or until chaos)      │
        └──────────────────────────┘
```

## Why?

Long tasks lose context. Claude forgets what it was doing 47 tool calls ago. Ralph fixes this by:

1. **Chunking work** - Inner Ralphs work on one unit at a time
2. **Reviewing progress** - Magi checks each chunk before continuing
3. **Persisting state** - Everything saved to file, so you can stop and resume

It's like having a responsible adult supervise the hyperactive code monkeys.

## Prerequisites

Works best with [magi](https://github.com/cbzehner/claude-skill-magi) installed for reviews. Falls back to self-review if unavailable (Ralph reviewing Ralph's work—*totally objective*).

## Installation

### From Marketplace

```bash
# Add the marketplace
/plugin marketplace add cbzehner/skill-ralph

# Install the skill
/plugin install ralph@cbzehner
```

### Manual Installation

```bash
cd ~/.claude/skills/
git clone https://github.com/cbzehner/skill-ralph.git ralph
```

## Usage

```
/ralph [state-file]
```

Point it at a plan or task file and watch the Ralphs go:

```
You: /ralph plans/auth-system.md

Claude: [Outer Ralph reads the plan]
        [Spawns Inner Ralph for Section 1]
        [Inner Ralph builds stuff, runs tests]
        [Inner Ralph returns: "I did things!"]
        [Magi reviews: "The things are acceptable."]
        [Outer Ralph updates plan, moves to Section 2]
        [Repeat until victory or /triple-ralph]
```

## Project Guidance (.ralph.md)

Ralph adapts to your project's conventions via a `.ralph.md` file at your repo root.

### What It Does

Without `.ralph.md`: Uses default plan-file conventions (YAML frontmatter + ## sections).

With `.ralph.md`: Adapts to your project's way of doing things:
- Where state files live
- How to identify work units (sections, acceptance criteria, issues)
- When and how to review
- How to signal completion

### Creating One

Run `/ralph` without a `.ralph.md` and you'll be offered the choice to create one. The skill asks questions about your project and generates a starter file.

Or see `examples/` for templates:
- `plan-based.ralph.md` - Traditional plan files with ## sections
- `task-based.ralph.md` - Task files with acceptance criteria (launchpad-style)
- `github-issues.ralph.md` - GitHub issues as work units
- `minimal.ralph.md` - Simplest possible configuration

See `examples/README.md` for a guide on crafting your own.

### Example .ralph.md

```markdown
# .ralph.md - My Project

## State Files
Tasks in `.tasks/` with YAML frontmatter and acceptance criteria.

## Work Units
Each unchecked criterion is a work unit. Group related criteria.

## Review
Self-review after each unit. Magi review before completion.

## Completion
When all criteria met: signal done via `/done`.
```

## Default State Format

When no `.ralph.md` is found, uses YAML frontmatter + markdown sections:

```markdown
---
status: pending  # pending | in_progress | complete | archived
gaps: []         # things we discovered we need
edge_cases: []   # things that might explode
progress: []     # sections completed
last_review: null
---

# My Feature Plan

## Section 1: The Setup
What to build...

## Section 2: The Hard Part
The actual work...
```

Frontmatter gets added automatically if missing.

## The Loop

```
┌─────────────────────────────────────────────────────────┐
│                     OUTER RALPH                         │
│                                                         │
│  1. LOAD    → Find .ralph.md, read state file           │
│  2. ASSESS  → Find next incomplete work unit            │
│  3. SPAWN   → Inner Ralph implements it                 │
│  4. REVIEW  → Magi (or self) checks the work            │
│  5. UPDATE  → Save progress per guidance                │
│  6. ROUTE   → Continue | Ask human | Complete           │
│                                                         │
│  Inner Ralph exits when:                                │
│  • Work unit complete                                   │
│  • Hit a blocker                                        │
│  • Max 20 turns (hard limit)                            │
│  • Context pressure                                     │
└─────────────────────────────────────────────────────────┘
```

## Completion

Work gets marked complete when:
1. Review says "this is done"
2. AND no gaps remain
3. AND per-guidance completion signals sent

Until then, the Ralphs keep Ralphing.

## Error Handling

| Problem | Solution |
|---------|----------|
| Magi unavailable | Ralph reviews Ralph (it's fine) |
| Inner Ralph blocked | Asks you for help |
| State file broken | Shows error, asks you to fix it |
| No .ralph.md | Offers to create one or uses defaults |
| Triple Ralph attempted | *inconceivable* |

## Files

```
ralph/
├── SKILL.md              # The actual skill
├── inner-prompt.md       # Template for Inner Ralph
├── README.md             # You are here
├── LICENSE               # MIT (Ralphs are free)
├── docs/
│   └── ARCHITECTURE.md   # The serious documentation
├── examples/
│   ├── README.md         # Guide for crafting .ralph.md
│   ├── plan-based.ralph.md
│   ├── task-based.ralph.md
│   ├── github-issues.ralph.md
│   ├── minimal.ralph.md
│   └── example-plan.md   # A sample plan file
└── plans/
    └── archived/         # Where completed plans go to rest
```

## Manual Override

Interrupt anytime. State is saved to file. Come back later and `/ralph` picks up where it left off. The Ralphs are patient.

## License

MIT

---

*"What could go wrong?"* — Everyone, right before finding out
