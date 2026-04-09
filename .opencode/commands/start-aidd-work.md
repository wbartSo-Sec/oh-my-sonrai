---
description: Start AIDD Story execution from SAR-ID plan
argument-hint: "[SAR-ID]"
agent: atlas
---

You are starting AIDD Story execution for a single SAR plan.

## WHAT TO DO

1. Parse the SAR-ID from `$ARGUMENTS`.

2. If no SAR-ID was provided:
   - Search `.sisyphus/plans/` for files matching `SAR-*.md`
   - List matches and ask the user to rerun with exactly one SAR-ID
   - Do not start execution until one SAR-ID is provided

3. Build the plan path as `.sisyphus/plans/$ARGUMENTS.md`.

4. Validate that `.sisyphus/plans/$ARGUMENTS.md` exists:
   - If missing, show a clear error
   - List available `.sisyphus/plans/SAR-*.md` files
   - Stop and wait for a valid SAR-ID

5. Create or update `.sisyphus/boulder.json` before any execution using this schema:

```json
{
  "active_plan": "<absolute-path-to-.sisyphus/plans/$ARGUMENTS.md>",
  "started_at": "<ISO_TIMESTAMP>",
  "session_ids": [],
  "plan_name": "$ARGUMENTS",
  "agent": "atlas"
}
```

6. Read `.sisyphus/plans/$ARGUMENTS.md` fully, then begin execution with Atlas orchestration workflow.

## CRITICAL

- Execute one Story at a time. Do not parallelize Stories within the same Epic.
- After each TODO completion, check Jira status transition requirements documented in the plan and apply required transitions.
- Verify feature flags for each task as part of QA before marking it done.
- Commands are prompt templates only. Do not attempt internal API calls.
- Use `.sisyphus/plans/$ARGUMENTS.md` as the execution plan path.
- Do not use `.ai/plans/` as an execution path.
- When you reach the Dev Lead approval gate in the plan, pause and ask the user for explicit approval before continuing.

## OUTPUT FORMAT

When SAR-ID is missing:

```text
Missing SAR-ID

Available plans in .sisyphus/plans/:
1. SAR-XXXX.md
2. SAR-YYYY.md

Rerun with: /start-aidd-work [SAR-ID]
```

When SAR-ID does not resolve to a plan:

```text
Plan Not Found

Expected: .sisyphus/plans/$ARGUMENTS.md
Available plans:
1. SAR-XXXX.md
2. SAR-YYYY.md

Provide a valid SAR-ID and rerun.
```

When execution starts:

```text
Starting AIDD Work

SAR-ID: $ARGUMENTS
Plan: .sisyphus/plans/$ARGUMENTS.md
Boulder state: .sisyphus/boulder.json updated

Reading plan and starting Atlas orchestration now.
```
