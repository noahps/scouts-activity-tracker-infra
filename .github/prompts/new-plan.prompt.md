---
description: Scaffold a new plan subfolder in .ai/plans/
---

I am about to start a new piece of work. Scaffold a new plan for it following
this project's plans convention.

Steps to take:
1. Ask me for the branch name I will be using (or suggest one based on my
   description using the format `YYYYMMDD_short-description`, where today's
   date is used for YYYYMMDD)
2. Create the folder `.ai/plans/<branch-name>/`
3. Copy `.ai/plans/PLAN_TEMPLATE.md` into `.ai/plans/<branch-name>/PLAN.md`
4. Fill in the Plan header fields (branch name, today's date, status: In progress)
5. Based on my description of the work, draft the Goal section and a first
   pass at the Steps checklist — leave the rest for me to complete
6. Add an entry to the Active plans section of `.ai/plans/README.md` in the
   format: `[branch-name](./branch-name/PLAN.md) — brief description`

Do not create the OUTCOME.md yet — that is written when the work is complete.

For more complex features that need a formal spec, suggest using Spec Kit
instead: `/speckit.specify` starts the structured spec → plan → tasks workflow.
