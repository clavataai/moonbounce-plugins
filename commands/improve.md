---
description: Improve a Moonbounce decision from its feedback, or resume it at the stage its brief records
argument-hint: <decision_id>
---

Run the improve flow, which improves a decision in use, for the Moonbounce decision `$ARGUMENTS`.

If no decision id was given, ask for one: a lowercase id such as `refund-exception`. Then use the `decisions` skill: read its "Flows" section and its improve guide, `references/improve.md`, and follow them.

Where the flow stands is the front matter of `moonbounce/$ARGUMENTS.brief.md`. If it records `flow: improve` with a `stage` that isn't `done`, resume at that stage. If it records `flow: build`, unfinished, ask before replacing it; at `stage: done` the decision is built, so start at `brief_check`, keeping the brief. If it records `flow: improve` at `stage: done`, start again at `brief_check`: improving a shipped decision is a new round. Otherwise start at `brief_check`, drafting a brief from the structure when there is none. Say which stage you are starting at and why before doing anything else.
