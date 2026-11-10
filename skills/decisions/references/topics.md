# Plan and procedure references

Generated from public topic metadata. This table is a lookup, not an execution order.

Read the returned plan. For a selected stage, use its `guidance.topic` with `plan_id: plan.id` in `get_decision_guidance`; the server resolves its pinned release. Check the returned plan/stage/topic/release/adapter against the plan. Follow role/shared links using `related_guidance[].arguments`, preserving that plan and owning stage.

| Plan stage | Reference | Purpose |
|---|---|---|
| `objective` | `plan.stages[id=objective].guidance` | Execute the objective procedure |
| `case_audit` | `plan.stages[id=case_audit].guidance` | Execute the case audit procedure |
| `reconcile` | `plan.stages[id=reconcile].guidance` | Execute the reconcile procedure |
| `brief` | `plan.stages[id=brief].guidance` | Execute the brief procedure |
| `probes` | `plan.stages[id=probes].guidance` | Execute the probes procedure |
| `label_check` | `plan.stages[id=label_check].guidance` | Execute the label check procedure |
| `cases` | `plan.stages[id=cases].guidance` | Execute the cases procedure |
| `first_version` | `plan.stages[id=first_version].guidance` | Execute the first version procedure |
| `baseline` | `plan.stages[id=baseline].guidance` | Execute the baseline procedure |
| `refine` | `plan.stages[id=refine].guidance` | Execute the refine procedure |
| `red_team` | `plan.stages[id=red_team].guidance` | Execute the red team procedure |
| `blind_check` | `plan.stages[id=blind_check].guidance` | Execute the blind check procedure |
| `import` | `plan.stages[id=import].guidance` | Execute the import procedure |
| `integration` | `plan.stages[id=integration].guidance` | Execute the integration procedure |

For a standalone task, retrieve `procedure.entry` to discover procedure topics, then use the resolved release explicitly. When a plan exists, retrieve `procedure.plan` with its ID and follow its stage references. Worker prompts are linked from stage guidance, not defined in this plugin.
