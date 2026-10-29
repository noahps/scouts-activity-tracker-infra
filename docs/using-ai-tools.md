# Using AI tools day-to-day in this project

## What's included

| Folder / file | Purpose |
|---|---|
| `AGENTS.md` | **Single source of truth** — all AI tools read this for project context |
| `CLAUDE.md` | Claude's context file — references `AGENTS.md`, adds Claude-specific notes |
| `.github/copilot-instructions.md` | Copilot's context file — references `AGENTS.md` |
| `.cursor/rules/index.mdc` | Cursor's rules file — references `AGENTS.md` |
| `.github/agents/` | Copilot agent definitions |
| `.github/prompts/` | Reusable `.prompt.md` files (usable by all tools) |
| `.github/PULL_REQUEST_TEMPLATE.md` | Standard PR description format |
| `.claude/commands/` | Claude Code custom slash commands |
| `.claude/skills/` | Spec Kit commands wired for Claude Code |
| `.claude/settings.json` | Project-scoped Claude Code plugins (currently: Superpowers, Matt Pocock Skills) |
| `.ai/plans/` | Branch-scoped work plans — one subfolder per branch, with `PLAN.md` and `OUTCOME.md` |
| `.ai/plans/PLAN_TEMPLATE.md` | Starting template for new `PLAN.md` files |
| `.ai/plans/OUTCOME_TEMPLATE.md` | Starting template for `OUTCOME.md` files written on completion |
| `.ai/specs/` | Design docs and longer-lived requirements |
| `.specify/` | Spec Kit working directory (generated specs, templates) |
| `.cursorignore` | Excludes generated files from Cursor's codebase index |
| `.editorconfig` | Consistent formatting across editors |

## Context file maintenance

`AGENTS.md` is the file to keep up to date. When your tech stack changes,
when conventions evolve, when the architecture shifts — edit `AGENTS.md`.
All four tools will pick up the changes automatically. The tool-specific files
(`CLAUDE.md`, `copilot-instructions.md`, `.cursor/rules/index.mdc`) only need
editing if you want to add or change something specific to that one tool.

## When to use Spec Kit

Use Spec Kit for features that benefit from a structured spec → plan → tasks
workflow — typically anything that will take more than a day or involves
meaningful design decisions:

```
/speckit.specify    — describe what to build (no tech stack yet)
/speckit.clarify    — AI asks clarifying questions
/speckit.plan       — add tech stack, generate implementation plan
/speckit.tasks      — break plan into ordered tasks
/speckit.implement  — agent executes the tasks
```

Artifacts are saved to `.specify/specs/<feature-name>/`. Any AI tool can
read these artifacts — you can spec with Copilot and implement with Cursor,
or plan with Claude and review with Antigravity.

## Plans workflow

Each piece of work gets its own subfolder in `.ai/plans/`, named to match the
git branch exactly (see more in [Plans README.md](../.ai/plans/README.md)):

```
.ai/plans/
├── README.md                             ← index of active + completed plans
├── PLAN_TEMPLATE.md                      ← copy this to start a new plan
├── OUTCOME_TEMPLATE.md                   ← fill this in when work is done
└── 20260515_refactor-login/
    ├── PLAN.md                           ← goal, steps, decisions (working doc)
    └── OUTCOME.md                        ← written on completion
```

**Starting new work:**
1. Create a branch: `git checkout -b 20260515_refactor-login`
2. Use the `new-plan` prompt or copy `PLAN_TEMPLATE.md` manually
3. Add an entry to `.ai/plans/README.md` under Active plans
4. Work through the plan, updating `PLAN.md` as understanding evolves

**Completing work:**
1. Write `OUTCOME.md` using `OUTCOME_TEMPLATE.md` — what happened, what
   changed, any gotchas
2. Move the entry in `.ai/plans/README.md` from Active to Completed
3. Commit both files alongside the code changes in the PR

**Simple changes on main:**

Not everything warrants a branch. For minor fixes, config tweaks, or
single-file edits committed directly to main, you can still use a plan folder
— just keep it lightweight:

1. Create the subfolder as usual: `YYYYMMDD_short-description/`
2. Write a minimal `PLAN.md` — goal + a short steps checklist is enough;
   skip Background and Decisions if they add no value
3. Skip `OUTCOME.md` unless something surprising happened worth recording
4. Add and immediately move the entry in `.ai/plans/README.md` from Active
   to Completed in the same commit

The folder name doesn't need to match a branch when there is no branch — use
a descriptive slug that makes the intent clear in the index.

**When to use Spec Kit instead:**

| Situation | Approach |
|-----------|----------|
| Quick fix, small task, exploratory spike | `.ai/plans/` manual plan |
| Feature taking more than ~1 day | Spec Kit (`/speckit.specify`) |
| Formal spec needed for review or handoff | Spec Kit (`/speckit.specify`) |

For large features, both can coexist: a Spec Kit spec in `.specify/specs/` for
formal structure, and a `.ai/plans/` entry for execution notes and outcomes.

## Superpowers plugin (Claude Code)

This project has the [Superpowers](https://github.com/obra/superpowers) plugin
installed (project scope, `.claude/settings.json`) — a skills framework by
Jesse Vincent that adds structured methodologies on top of Claude Code:

- **Brainstorming** (`/brainstorming`) — Socratic, divergent exploration of a
  problem before committing to an approach
- **TDD** — enforced red-green-refactor cycles (failing test → minimal
  implementation → refactor)
- **Systematic debugging** — a four-phase root-cause methodology, with a
  safeguard that forces an architectural review after three failed fix attempts
- **Subagent-driven development** (`/execute-plan`) — implementation with a
  built-in code-reviewer agent

Superpowers and Spec Kit operate at different layers and can be used together.

| Phase | Tool | Output |
|---|---|---|
| Explore the problem, weigh approaches | Superpowers `/brainstorming` | A chosen direction (disposable notes — don't keep this as a parallel decision record) |
| Turn the direction into a structured spec | Spec Kit `/speckit.specify` → `/speckit.plan` → `/speckit.tasks` | `spec.md`, `plan.md`, `tasks.md` in `.specify/specs/` |
| Implement each task | Superpowers TDD (red-green-refactor) via `/speckit.implement` or `/execute-plan` | Working code, tested |
| Something breaks during implementation | Superpowers debugging methodology | Root-cause fix |

The key rule: only one artifact should own "what we decided and why" for a
given feature. Once a brainstorm concludes, fold its outcome into `spec.md` /
`plan.md` rather than keeping a separate Superpowers write-up — Spec Kit's
artifacts (plus `.ai/plans/`) stay the source of truth for this repo.

## Matt Pocock Skills plugin (Claude Code)

This project also has the [Matt Pocock Skills](https://github.com/mattpocock/skills)
plugin installed (project scope, `.claude/settings.json`) — a second skills
collection covering engineering and productivity workflows:

- **Engineering** — `/tdd`, `/code-review`, `/diagnosing-bugs`,
  `/domain-modeling`, `/codebase-design` (model-invoked), plus
  `/triage`, `/to-spec`, `/to-tickets`, `/implement`, `/wayfinder`,
  `/improve-codebase-architecture` (user-invoked)
- **Productivity** — `/grill-me` (relentless planning interview),
  `/handoff` (conversation compaction), `/teach`, `/to-questionnaire`

Its skill names are namespaced (e.g. `mattpocock:code-review`), so they don't
collide with Superpowers or Claude's built-ins — but the workflows overlap
conceptually, so this repo's convention is:

| Concern | Owned by |
|---|---|
| Spec/plan/tasks source of truth | Spec Kit (`spec.md`, `plan.md`, `tasks.md` in `.specify/specs/`) |
| Brainstorming, TDD execution, debugging during implementation | Superpowers |
| Turning specs into tracked tickets, backlog triage | Matt Pocock Skills (`/to-tickets`, `/triage`) — no equivalent in the other two |

Where more than one plugin offers the same capability (TDD, code review),
pick one as the project convention and record the choice in `AGENTS.md` —
this doc doesn't hardcode it.

This plugin has a required per-project setup step (unlike Superpowers): run
`/setup-matt-pocock-skills` once to answer which issue tracker, triage
labels, and doc paths this project uses. It writes that config under
`docs/agents/`, which should then be committed. See the
[README](../README.md#after-cloning--per-project-setup) for the full
per-project setup sequence, including the explicit plugin install.

## Switching between tools

All four tools share the same context (`AGENTS.md`) and the same artifact
files on disk. The markdown files are the handoff mechanism — any tool can
pick up where another left off.

| Switching from → to | What carries over |
|---|---|
| Claude → Copilot | `AGENTS.md`, `.ai/plans/`, `.specify/specs/` |
| Cursor → Claude Code | `AGENTS.md`, `.ai/plans/`, `.specify/specs/` |
| Any tool → Antigravity | `AGENTS.md` (Antigravity reads it natively) |
