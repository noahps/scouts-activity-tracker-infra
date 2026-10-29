# Scouts Activity Tracker Infra

> This file is the single source of truth for AI context across all tools
> (Claude, Copilot, Cursor, Antigravity). Remaining placeholder sections
> (marked with `[...]`) are still to be filled in as the project takes shape.

## Project overview

The infrastructure and GraphQL API for Scouts Activity Tracker, a single-page app for
tracking and managing scouting activities for a local scouts group. This repo
defines everything the app runs on in AWS, as code, and the GraphQL schema (the
API contract) the app talks to.

The app this repo supports will:

- Track activities and the linkages between those activities and Scouts Canada
  badges.
- Track participation in those activities by leaders and by individual
  Beavers, Cubs and Scouts.
- Track each youth member's progress toward badges, based on their activity
  participation.

The front-end app lives in a separate repo, `scouts-activity-tracker`, expected
in a sibling folder (`../scouts-activity-tracker`). It is built with Vue 3
and Vuetify, with i18n (EN first, FR later). The GraphQL schema
defined in this repo is the source of truth for the data models and operations
the front end consumes, so changes to it must be made with the front end in
mind.

## Tech stack

| Layer | Technology |
|-------|-----------|
| Cloud | AWS |
| Infrastructure as code | Terraform (state in `ursudio-terraform-backend`) |
| Automation | `make` (Makefiles) wrapping Terraform and the AWS CLI |
| API definition | GraphQL schema (`.graphql`), served by AWS AppSync |
| Front end | `scouts-activity-tracker` (separate repo, sibling folder) |
| Lambda runtime | [TBD] |
| Key dependencies | Terraform AWS provider, AWS CLI, GNU Make, [TBD] |

### Initial AWS services

| Area | Services |
|------|----------|
| Identity | Cognito |
| Edge, DNS and TLS | CloudFront, Route 53, certificates (ACM) |
| API | AppSync (GraphQL queries, mutations and subscriptions) |
| Compute | Lambda |
| Data and storage | DynamoDB, S3 |
| AI | Bedrock |

- **Bedrock** assists with activity plan creation and with analyzing activities
  against Canadian Path badge requirements.
- **AppSync** is the single API: queries, mutations and subscriptions over
  GraphQL. Subscriptions provide real-time updates so multiple users see status
  changes instantly. For example, when a leader marks a cub as present, other
  leaders see the updated status in their view of the app. Resolvers talk to
  DynamoDB directly or via Lambda (e.g. for Bedrock calls). Prefer JavaScript
  resolvers (APPSYNC_JS) over VTL.

This is the expected starting set, not a closed list. Record additions and the
reason for them here as the architecture evolves.

### Technical recommendation: production code should use established frameworks

Default guidance for this repo. Keep it, tighten it into a requirement, or
record an agreed exception:

- **Lambda code**: production Lambda code is highly encouraged to be built on
  established libraries/frameworks for its language rather than plain/vanilla
  code, and to be unit tested.
- **Prototyping exception**: plain scripts are fine for fast prototypes and
  experiments (e.g. under `research/experiments/`). That code isn't
  production code and shouldn't be promoted as-is. Rebuild it properly when it
  graduates to production.
- **Infrastructure**: anything significant that gets deployed should be
  defined as infrastructure-as-code (IaC), not configured by hand in a
  console. This repo uses Terraform, with state held in the
  `ursudio-terraform-backend`. See that backend's README for how to add a
  project to it.
- **Production code standards**: once code is designated production (not
  prototype/experiment), it should follow established industry best
  practices: test suites, documentation, and issue tracking, not just
  working code.

## Repository structure

```
[Describe the main folders and what lives in them once the repo is
populated — e.g. terraform/ (modules and environments), graphql/ (schema),
lambdas/ (function code), tests/, docs/, Makefile]
```

The front-end app lives in a separate repo (`scouts-activity-tracker`),
expected as a sibling folder. Don't copy front-end code into this repo.

## How to run

All routine tasks run through `make`. Makefile targets wrap a combination of
Terraform and the AWS CLI. AWS CLI commands use the `tatascouts` profile
(`--profile tatascouts`); Terraform should use the same profile (e.g. via
`AWS_PROFILE`). Don't hard-code credentials, and don't run raw `terraform` or
`aws` commands for routine work when a make target exists. Add a target
instead.

```bash
# List available targets
make help

# Initialise Terraform (configures the ursudio-terraform-backend state)
make init

# Validate and format
make validate
make fmt

# Plan changes (review before applying)
make plan

# Apply changes
make apply

# Lint (Terraform, GraphQL schema, Lambda code)
make lint

# Run tests (see "Docs, testing, validation and linting")
make test

# Check the schema for breaking changes against main
make schema-check

# Generate docs (schema docs, Terraform module docs)
make docs
```

The targets above are the intended interface and don't exist yet. Update this
section to match the Makefile as it is built.

## Coding conventions

- Routine operations go through `make` targets, which use Terraform and the
  AWS CLI with `--profile tatascouts`. Keep targets documented (`make help`)
- All AWS resources are defined in Terraform; no manual console changes. If
  a console change is unavoidable, record it and bring it back into code
- The GraphQL schema (`graphql/schema.graphql`) is the source of truth for
  the API. AppSync is deployed from it, not configured separately
- Prefer small, reusable Terraform modules; pin provider and module versions
- Name and tag resources consistently (project, environment)
- Least-privilege IAM for every Lambda, role and policy; no wildcard
  permissions without a recorded reason
- Authorize every GraphQL operation through Cognito (user pool auth); don't
  add public/unauthenticated operations without a recorded reason
- [Tests live alongside source files — confirm when test tooling is chosen]

## Docs, testing, validation and linting

These are the planned practices for this repo. Confirm tool choices as the
first modules are built.

- **Docs generation.** Require descriptions (`"""doc strings"""`) on every
  type, field and operation in the schema, and generate the schema docs from
  it (e.g. SpectaQL or graphql-markdown) rather than writing them by hand.
  Generate Terraform module docs with `terraform-docs`. Generated docs are
  rebuilt by `make docs`, not hand-edited.
- **Testing.**
  - Terraform: `terraform fmt -check`, `terraform validate`, and native
    `terraform test` for modules. Review every `terraform plan`.
  - Schema: GraphQL Inspector (or equivalent) to diff the schema against
    `main` and fail on breaking changes (`make schema-check`).
  - Resolvers: unit test APPSYNC_JS resolvers (e.g. with the AppSync
    `EvaluateCode` API / `aws appsync evaluate-code`) and Lambda code with a
    standard test runner.
  - Integration: run operations against a deployed dev stack for auth,
    resolvers, DynamoDB access and subscriptions.
- **Request validation.** AppSync already rejects operations that don't match
  the schema, so lean on the type system: required (`!`) fields, enums, input
  types and AppSync scalars (e.g. `AWSDate`, `AWSEmail`). Add further
  validation (lengths, ranges, business rules) in resolvers or Lambda and
  return clear errors. Enforce authorization per operation. Set the API's
  query depth and resolver count limits. Consider AWS WAF on the AppSync API
  for rate limiting.
- **Linting and security scanning.** `terraform fmt` and `tflint` (with the
  AWS ruleset), a Terraform security scanner (e.g. Checkov or Trivy),
  `@graphql-eslint` for the schema (descriptions, naming conventions), and
  ESLint/Prettier for Lambda code. All run through `make lint` and in CI.

## Portability and vendor neutrality

AWS is the current target, but the project should stay understandable and
reusable beyond it. The Terraform is useful only to other AWS users, but other
developers can read the schema and data model and build their own back end.

- **Schema is the contract.** Keep a plain `.graphql` file describing the
  data model. It is the contract the front end (`scouts-activity-tracker`)
  codes against, so keep it free of AWS-specific directives where practical
  and document it in this repo.
- **Standard OIDC.** Cognito is used as a standard OIDC provider; the front
  end relies on OIDC, not Cognito-specific SDK behaviour.
- **No secrets, state or account IDs in the repo.** Don't commit secrets,
  Terraform state files, or AWS account IDs.
- **Note on Amplify.** Amplify Gen 2 Data deploys with CDK, not Terraform.
  AppSync, DynamoDB and Cognito are defined here with Terraform resources, so
  the front end cannot use Amplify model helpers and uses plain GraphQL
  queries.

Open-source checklist:

- [ ] Choose a license for the repo
- [ ] README has setup steps and the schema contract
- [ ] Confirm nothing committed contains secrets, state or account IDs

## Always / never

**Always:**
- Write a test for every new function
- Update AGENTS.md when architecture changes
- Run `terraform plan` and review it before any apply
- Keep the GraphQL schema and the front end's expectations in sync, and run
  `make schema-check` before changing it

**Never:**
- Commit directly to main
- Store credentials or secrets in code, or commit Terraform state, AWS
  account IDs, or `.tfvars` files containing secrets
- Apply Terraform or change AWS resources without explicit human approval
- Put real member data (names, ages, contact details of youth) in code,
  fixtures, plans or prompts. Use obviously fake sample data

## AI usage rules (applies to all agents)

**Always:**
- Stop and ask before reading, pasting, or sending anything that looks like
  confidential or personal information: member or youth data, credentials,
  or other personal details. When in doubt, data stays out.
- Treat your output as a draft that a human must verify. Flag assumptions,
  uncertainties, and unverified facts or figures instead of stating them
  confidently.
- Get explicit human approval before any action with external side effects
  (sending, publishing, pushing, writing to external systems, spending).
- Read any third-party skill, plugin, or MCP server before running it or
  suggesting it be added. Don't add new AI tools, plugins, or MCP servers
  without the user's confirmation.
- Keep AI co-author attribution on commits, and note significant AI
  contribution in PR descriptions.
- Prefer the simplest effective approach. Use direct edits and
  deterministic scripts over repeated LLM iterations or unnecessary
  subagents.

**Never:**
- Invent requirements, badge rules or Scouts Canada program details.
  Surface options and let a person decide.
- Include confidential or personal data in prompts to external services,
  commit messages, plans, logs, or artifacts.
- Present AI-generated output as verified or as final human work.

## Active work

Work is tracked in `.ai/plans/`. Each piece of in-flight work has its own
subfolder named `YYYYMMDD_short-description`, which matches the git branch
name exactly.

Each subfolder contains:
- `PLAN.md` — goal, decisions, constraints, and ordered steps (working document)
- `OUTCOME.md` — written on completion: what happened, what changed, gotchas

The index of active and completed plans is in `.ai/plans/README.md`.

For features requiring a formal spec, Spec Kit artifacts live in
`.specify/specs/` and follow their own structure.

**To find the current work:** check `.ai/plans/README.md` for active plans,
or look for a subfolder matching the current branch name.

**To start new work:** create a subfolder matching your branch name, copy
`.ai/plans/PLAN_TEMPLATE.md` to `PLAN.md` inside it, and add an entry to
`.ai/plans/README.md`. Or use the `/new-plan` prompt.

## AI tooling in this repo

- **Claude** — reads `CLAUDE.md`, which references this file
- **Copilot** — reads `.github/copilot-instructions.md`, which references this file
- **Cursor** — reads `.cursor/rules/index.mdc`, which references this file
- **Antigravity** — reads this file directly
- **Spec Kit** — slash commands in `.claude/skills/` (Claude Code) and `.github/` (Copilot)

See `README.md` for how to use the template and start new projects.
