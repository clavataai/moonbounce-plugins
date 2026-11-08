---
description: Build a Moonbounce decision from a policy, labeled data or a description, or resume it at the stage its brief records
argument-hint: <decision_id>
---

Run the build flow, which builds a new decision from a policy, labeled data or a description, for the Moonbounce decision `$ARGUMENTS`.

If no decision id was given, ask for one: a lowercase id such as `refund-exception`. Then use the `decisions` skill: read its "Flows" section and its build guide, `references/build.md`, and follow them.

Where the flow stands is the front matter of `moonbounce/$ARGUMENTS.brief.md`. If it records `flow: build` with a `stage` that isn't `done`, resume at that stage. If it records `flow: improve`, at any stage, ask before replacing it. If it records `stage: done`, the decision is built: ask whether the person wants to start over, which rewrites the brief, or improve it from feedback (`/moonbounce:improve`). Otherwise start at `objective`. Say which stage you are starting at and why before doing anything else.
