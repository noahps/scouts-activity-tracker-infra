<!--
SYNC IMPACT REPORT
Version change: (template, unversioned) → 1.0.0
Modified principles: none (initial adoption; all placeholders filled)
Added sections: Core Principles I–VI, Technology & Operational Constraints,
  Development Workflow & Quality Gates, Governance
Removed sections: none
Templates requiring updates:
  - ✅ .specify/templates/tasks-template.md — tests changed from OPTIONAL to REQUIRED
    (edited at the maintainer's explicit request, despite being Spec Kit-managed).
  - ⚠ pending .specify/templates/plan-template.md — "Constitution Check" gates are a
    placeholder; /speckit-plan must derive gates from Principles I–VI.
  - ✅ spec-template.md — no conflicts.
  - ✅ AGENTS.md / CLAUDE.md — consistent; no changes needed.
Follow-up TODOs:
  - TODO(RATIFICATION_DATE): set to the date of first adoption (2026-10-28); change if the
    team formally adopted it earlier.
  - AGENTS.md still lists the Lambda runtime as [TBD]; update it to match the new rule.
-->
# Scouts Activity Tracker Infra Constitution

## Core Principles

### I. Everything as Code, Through Make
All AWS resources MUST be defined in Terraform; manual console changes are prohibited. If
one is unavoidable it MUST be recorded and brought back into code. Routine operations MUST
run through documented `make` targets (visible in `make help`) that use the `tatascouts`
AWS profile; when no target exists, one MUST be added rather than running raw `terraform`
or `aws` commands. Provider and module versions MUST be pinned, and resources MUST be named
and tagged consistently (project, environment). Rationale: reproducibility and auditability.

### II. GraphQL Schema Is the Contract
`graphql/schema.graphql` is the source of truth for the API; AppSync is deployed from it,
not configured separately. Every type, field and operation MUST have a description. The
schema MUST stay free of AWS-specific directives where practical. Before any schema
change, `make schema-check` MUST be run to detect breaking changes against `main`, and the
change MUST be made with the front end (`scouts-activity-tracker`) in mind. Rationale:
the front end codes against this file and other developers may reuse it.

### III. Test Every New Function (NON-NEGOTIABLE)
Every new function MUST have a test. APPSYNC_JS resolvers and Lambda code MUST be unit
tested; Terraform modules MUST pass `terraform fmt -check`, `terraform validate` and
`terraform test`; schema changes MUST pass `make schema-check`. Authentication, resolvers,
DynamoDB access and subscriptions MUST be verified against a deployed dev stack.
Rationale: infrastructure mistakes are costly and hard to reverse.

### IV. Secure by Default
Every GraphQL operation MUST be authorized through Cognito user pool auth; public or
unauthenticated operations require a recorded reason. IAM MUST be least-privilege for every
Lambda, role and policy; wildcard permissions require a recorded reason. Inputs MUST be
validated using the schema type system (required fields, enums, input types, AppSync
scalars) plus resolver or Lambda checks for business rules, with clear errors. API query
depth and resolver count limits MUST be set. Terraform MUST pass `tflint` and a security
scanner via `make lint` and CI. Rationale: the system holds data about children.

### V. Protected Data and Secrets
Secrets, Terraform state, AWS account IDs and `.tfvars` files containing secrets MUST NOT
be committed. Real member data (names, ages, contact details of youth) MUST NOT appear in
code, fixtures, plans, prompts, logs or commit messages; use obviously fake sample data.
Credentials MUST NOT be hard-coded. Rationale: the repo may be open-sourced.

### VI. Human-Approved Change and Portability
`terraform plan` MUST be reviewed before any apply, and Terraform MUST NOT be applied nor
AWS resources changed without explicit human approval. Auth MUST remain standard OIDC
(Cognito as an OIDC provider, no reliance on Cognito-specific SDK behaviour) and AppSync
resolvers SHOULD use APPSYNC_JS rather than VTL. Rationale: keep deployments deliberate and
the API reusable with other identity providers and front ends.

## Technology & Operational Constraints

- Lambda runtime: Node.js or Python, chosen per task. Each function MUST use one of the two
  with an established test runner for that language (e.g. Vitest/Jest for Node, pytest for
  Python) and the language's standard linter, run through `make lint` and `make test`.
- Stack: AWS, Terraform (state in `ursudio-terraform-backend`), GNU Make, AWS CLI, AppSync,
  Cognito, DynamoDB, Lambda, S3, CloudFront, Route 53, ACM, Bedrock. Additions and their
  reasons MUST be recorded in AGENTS.md.
- This repo contains no front-end code; the front end lives in `scouts-activity-tracker`.
- Generated docs (schema docs, `terraform-docs` output) MUST be produced by `make docs`
  and never hand-edited.
- Amplify Gen 2 Data (CDK) is incompatible with this Terraform setup; the front end uses
  plain GraphQL.
- Prototype code (e.g. `research/experiments/`) MUST NOT be promoted to production as-is.

## Development Workflow & Quality Gates

- Work MUST NOT be committed directly to `main`; use a branch whose name matches its
  `.ai/plans/YYYYMMDD_branch-name/` folder, containing `PLAN.md` and, on completion,
  `OUTCOME.md` (with the `.ai/plans/README.md` index updated).
- AGENTS.md MUST be updated when architecture changes.
- AI agents MUST NOT invent requirements, badge rules or Scouts Canada program details;
  AI output is a draft for human verification, and external side effects (including
  apply, push, publish) require explicit human approval.
- AI co-author attribution MUST be kept on commits, and significant AI contribution noted
  in PR descriptions.
- Production code MUST ship with tests, documentation and tracked issues. Every PR and
  review MUST verify compliance with this constitution.

## Governance

This constitution supersedes other practices in this repo; AGENTS.md carries runtime
development guidance and MUST remain consistent with it. Amendments require a pull request
that updates this file, includes a Sync Impact Report, and updates dependent templates or
docs (or records them as pending). Versioning follows semantic versioning: MAJOR for
backward-incompatible principle removals or redefinitions, MINOR for added principles or
materially expanded guidance, PATCH for clarifications and wording. Deviations from a
principle MUST be justified in the plan's Complexity Tracking table and approved by the
maintainer. Compliance is reviewed at every PR and re-checked at plan time through the
plan's Constitution Check.

**Version**: 1.0.0 | **Ratified**: 2026-10-28 | **Last Amended**: 2026-10-28
