---
description: Build a Moonbounce decision from a policy (flow 1), or resume it at the stage its brief records
argument-hint: <decision_id>
---

Run flow 1, build a decision from a policy, for the Moonbounce decision `$ARGUMENTS`.

If no decision id was given, ask for one: a lowercase id such as `refund-exception`. Then use the `decisions` skill: read its "Flows" section and its build guide, `references/build.md`, and follow them.

Where the flow stands is the front matter of `moonbounce/$ARGUMENTS.brief.md`. If it records `flow: build` with a `stage` that isn't `done`, resume at that stage. If it records `flow: improve`, unfinished, ask before replacing it. Otherwise start at `objective`. Say which stage you are starting at and why before doing anything else.
