---
description: Start or resume a decision with conversational intake and an advisory plan
argument-hint: [decision_id] [what you want to do]
---

Use the `decisions` skill's `references/intake.md` procedure for `$ARGUMENTS`.

Read `get_decision_intake`, establish create, revise, assess or investigate from the user's request, inspect supplied materials and reuse known facts. Do not infer the operation merely from the existence of a decision file. Use a known decision ID from the request or artifacts; clarify an ambiguous target before writing.

Write and upload the intake, then call `create_decision_plan` with the received upload ID. Read an existing plan and verify its artifacts when resuming. Follow the selected stage procedures through the adapter, preserving remaining budgets. Missing tools or guidance are requirements to report, not a reason to run another workflow. Assessment never edits the candidate or settings; authoring alone never registers or scores.

Preserve the requested endpoint and stopping point in the intake task. A quick, scored first version requires actual measurement, not an untested draft; perform registration/objective setup only when the selected score or use requires it and the operation authorizes it. Verify the report's actual objective; disclose balanced-loss scoring when no custom objective is stored. Stop after the requested report without automatically adding tuning, later checks, import or app wiring. Disclose any deferred evidence goals.
