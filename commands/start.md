---
description: Start or resume work on a Moonbounce decision; picks build (flow 1) or improve (flow 2) from what you ask for and what exists
argument-hint: [decision_id] [what you want to do]
---

Start or resume work on a Moonbounce decision: `$ARGUMENTS`.

Use the `decisions` skill and read its "Flows" section. If no decision id was given, ask for one: a lowercase id such as `refund-exception`. Then pick the flow by the skill's rules:

1. If the request says what to do (build a decision from a policy, or improve one from its feedback), run that flow. If the brief records the other flow, unfinished, ask before replacing it.
2. Otherwise, if `moonbounce/<decision_id>.brief.md` records a `stage` that isn't `done`, resume that flow at that stage.
3. Otherwise, if the decision has labeled cases in the key's environment (`list_cases`), start flow 2, improve, at `brief_check`.
4. Otherwise, start flow 1, build, at `objective`.

Say which flow and stage you picked and why before doing anything else, so the person can redirect you. Then follow that flow's guide: `references/build.md` or `references/improve.md`.
