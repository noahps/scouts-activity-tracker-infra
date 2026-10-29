# Scouts Activity Tracker Infra

The AWS infrastructure and GraphQL API for Scouts Activity Tracker, a single-page app
for tracking and managing scouting activities for a local scouts group.

The app it supports lets leaders:

- Track activities and the linkages between those activities and Scouts Canada
  badges.
- Track participation in those activities by leaders and by individual
  Beavers, Cubs and Scouts.
- Track each youth member's progress toward badges, based on their activity
  participation.

## Tech stack

- [AWS](https://aws.amazon.com/)
- [Terraform](https://www.terraform.io/), with state in the
  `ursudio-terraform-backend`
- [GNU Make](https://www.gnu.org/software/make/) targets wrapping Terraform
  and the [AWS CLI](https://aws.amazon.com/cli/) (profile `tatascouts`)
- [GraphQL](https://graphql.org/) schema served by AWS AppSync as the API
  definition

Initial AWS services: Cognito, S3, ACM certificates, Route 53,
AppSync, CloudFront, Lambda, DynamoDB and Bedrock. Bedrock helps
with activity plan creation and analyzing activities against Canadian Path
badge requirements. AppSync provides real-time updates so that, for example, when
a leader marks a cub as present, other leaders see it immediately.

## Related repos

The front-end app is maintained in
[`scouts-activity-tracker`](https://github.com/noahps/scouts-activity-tracker),
which is expected to live in a sibling folder:

```
websites/
├── scouts-activity-tracker/         # front end (Vue 3, Vuetify)
└── scouts-activity-tracker-infra/   # this repo (infrastructure and GraphQL API)
```

The GraphQL schema in this repo is the source of truth for the API the front
end consumes.

## Portability

AWS is the current target, but the project is meant to stay vendor-neutral
where practical. The Terraform here is useful to other AWS users only, but
other developers can read the GraphQL schema and data model and build their
own back end. The schema is the contract with the front end. Cognito is used
as a standard OIDC provider, and no secrets, Terraform state or account IDs are
committed to this repo.

Setup steps and the schema contract are still to be written, and a license
is still to be chosen.

## Getting started

This repo is still being set up. Day-to-day work will run through `make`
targets that wrap Terraform and the AWS CLI, using the `tatascouts` AWS profile.
Prerequisites, the make targets and deployment notes will be added here once the
first modules exist.

## AI tooling

See [AGENTS.md](AGENTS.md) for AI tooling context (used by Claude, Copilot,
Cursor, and Antigravity).

See [docs/using-ai-tools.md](docs/using-ai-tools.md) for how to use the AI
tools day-to-day (Spec Kit, `.ai/plans/`, Superpowers, Matt Pocock Skills, and
switching between tools).

### One-time setup per clone

```bash
# Run the Spec Kit constitution once per project
# (open the project in Claude Code or Copilot and run)
/speckit.constitution

# Install the Matt Pocock Skills plugin (Claude Code). It's declared in
# .claude/settings.json but doesn't reliably auto-fetch, so install it
# explicitly from the repo root:
claude plugin install mattpocock-skills@claude-plugins-official --scope project -y

# Then run its setup once to choose the issue tracker, triage labels and
# doc paths (writes config under docs/agents/):
/setup-matt-pocock-skills
```

The Superpowers plugin is also declared in `.claude/settings.json` and is
fetched automatically the first time the project is opened in Claude Code
(accept the one-time marketplace trust prompt). If it isn't picked up, sync it
manually from the repo root:

```bash
claude plugin marketplace update
claude plugin install superpowers@superpowers-dev --scope project -y
```

The Spec Kit CLI (`specify`) is optional; Spec Kit ships vendored in this repo.
Install it only to add an integration for another tool or to upgrade the
vendored templates:

```bash
uv tool install specify-cli
specify init --here --ai <copilot|claude|cursor>
specify self check
specify self upgrade
```
