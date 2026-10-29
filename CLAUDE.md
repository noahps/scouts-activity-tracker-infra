# Claude project context

Scouts Activity Tracker Infra: the AWS infrastructure and GraphQL API for Scouts
Activity Tracker, a single-page app for tracking scouting activities, badge
linkages and youth progress for a local scouts group. Built with Terraform,
`make` and a GraphQL schema on AWS (AWS CLI profile `tatascouts`), using Cognito,
S3, ACM certificates, Route 53, AppSync, CloudFront, Lambda,
DynamoDB and Bedrock. The front end is the separate `scouts-activity-tracker`
repo (sibling folder).

> The primary project context for this repo is in `AGENTS.md`.
> Please read that file first for project overview, tech stack, conventions,
> and active work tracking.

## Claude-specific notes

- Custom slash commands for this project are in `.claude/commands/`
- Spec Kit commands are available via `.claude/skills/` — use `/speckit.specify`,
  `/speckit.plan`, `/speckit.tasks`, `/speckit.implement` for structured feature work
- Do not edit files in `.specify/memory/`, `.specify/templates/`, or
  `.specify/scripts/` — these are managed by Spec Kit
- Plans for in-flight work live in `.ai/plans/` — each branch has a matching
  subfolder named `YYYYMMDD_branch-name/` containing `PLAN.md`
- When starting work, check `.ai/plans/README.md` for the active plan or look
  for a subfolder matching the current branch name
- When completing work, write `.ai/plans/YYYYMMDD_branch-name/OUTCOME.md`
  using `.ai/plans/OUTCOME_TEMPLATE.md` as a starting point, then update the
  index in `.ai/plans/README.md`
- Use `.ai/plans/PLAN_TEMPLATE.md` as a starting point for new plans, or use
  the `.github/prompts/new-plan.prompt.md` prompt to scaffold the full structure

<!-- SPECKIT START -->
For additional context about technologies to be used, project structure,
shell commands, and other important information, read the current plan
<!-- SPECKIT END -->

## Agent skills

### Issue tracker

Issues live in GitHub Issues (`noahps/scouts-activity-tracker-infra`), via the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

Default vocabulary: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `GLOSSARY.md` + `docs/adr/` at the repo root. See `docs/agents/domain.md`.
