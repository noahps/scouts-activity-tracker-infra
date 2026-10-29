# .ai/plans

This folder tracks all in-flight and completed work plans for Scouts Activity
Tracker Infra, the AWS infrastructure (Terraform) and GraphQL API (AppSync) that
supports the Scouts Activity Tracker front end. See `AGENTS.md` for the full
project context.

## Convention

Each piece of planned work gets its own subfolder. The folder name follows this
format:

```
YYYYMMDD_short-description-of-work/
```

The date is when the work was started. The slug is kebab-case and should match
the git branch name exactly — this is the link between the plan and the code.

Each subfolder contains:

- `PLAN.md` — written before or at the start of work. Describes the goal,
  key decisions, constraints, and ordered steps. This is a working document:
  update it as understanding evolves during execution.
- `OUTCOME.md` — written when the work is merged or abandoned. Describes what
  actually happened, what changed from the plan, any gotchas or decisions made
  during execution. Valuable context for future work in the same area.

Use `PLAN_TEMPLATE.md` and `OUTCOME_TEMPLATE.md` in this folder as starting
points. The `.github/prompts/new-plan.prompt.md` prompt will scaffold a new
plan subfolder automatically.

## When to use this vs Spec Kit

| Situation | Use |
|-----------|-----|
| Quick fix, small task, exploratory spike | `.ai/plans/` manual plan |
| Feature taking more than a day | Spec Kit (`/speckit.specify`) |
| Formal spec needed for review or handoff | Spec Kit (`/speckit.specify`) |

Spec Kit artifacts live in `.specify/specs/` and follow their own structure.
For large features, you may have both: a Spec Kit spec for the formal structure
and an `.ai/plans/` entry for day-to-day execution notes.

## Active plans

> Update this section when starting or completing work.
> Format: `[branch-name](./folder-name/PLAN.md) — brief description`

_No active plans._

## Completed plans

> Move entries here when the branch is merged. Add a link to the OUTCOME.md.
> Format: `[branch-name](./folder-name/OUTCOME.md) — brief description`

_No completed plans yet._
