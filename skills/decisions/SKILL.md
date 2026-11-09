---
name: decisions
description: "Use when calling the Moonbounce Decisions MCP or API: build a decision from a policy, labeled data or a description (the build flow) or improve one in use (the improve flow), resuming where the brief says; define/register/evaluate a decision, read back or list decisions, versions, evaluations and labeled cases, import labeled examples or send feedback, score a dataset, backtest, set the objective, pick a model, check cost."
---

# Moonbounce Decisions

A decision is a JSON **structure**: `questions` a model answers over the call's `state` (each may list the `state.<key>` paths it `reads`, nested ones like `state.order.total` included), optional `attributes` the caller supplies, `aggregations` that combine them (weights, thresholds and cut points written in), and a `result` naming the final feature. Every reference inside a structure is a path: `questions.<name>`, `attributes.<name>` or `aggregations.<name>` (`"result": "aggregations.handling"`); a bare name is a 400. The caller's code owns the structure; Moonbounce runs it, records evaluations and learns from feedback. Imported examples, feedback, review and previews all take this shape.

## The smallest call

Two calls that store nothing show the connection works. Use the tools, or REST with `Authorization: Bearer <key>`:

1. `whoami` (`GET /v1/whoami`) returns the org, environment and scopes behind the key or sign-in. It reads nothing and spends nothing.
2. A dry-run `evaluate` (`POST /v1/evaluate`) registers and stores nothing, spends one model call, and answers with `status`, `answers`, the `version` that ran (`number` is `null` while that structure isn't registered yet; a dry run registers nothing) and `cost`:

```json
{
  "state": { "message": "The zipper on my tent broke the first night. Can I return it?" },
  "questions": { "defective": { "type": "noul", "instructions": "Does the customer describe a defect or damage?" } },
  "result": "questions.defective",
  "moonbounce": { "decision_id": "skill.smoke", "item_id": "t-001", "dry_run": true }
}
```

Everything else here builds on that call: the structure grows aggregations and cut points, and dropping `dry_run` stores the evaluation. Then run the discovery routine below.

## Discovery routine — first, every session

1. Use the connected `moonbounce` MCP server (with the Claude Code plugin it appears as `plugin:moonbounce:moonbounce`). If it's listed but its tools aren't available because it needs sign-in, you can't start sign-in yourself: ask the person to run `/mcp` → the server → **Authenticate** (or `claude mcp login <name>` in a shell, for a server they added by hand), then continue. If none, connect your agent to `https://api.moonbounce.ai/mcp` by sign-in or key; see [per-agent installation](references/install.md). Sign-in gives admins and members `decisions:develop` and `decisions:evaluate` in Production, including real and dry-run evaluations and reads, attributed to the person. Unattended agents, backends, Test (`env:test`) and REST need an API key with `decisions:develop`/`decisions:evaluate` as needed; send it as `Authorization: Bearer <key>`. Never print the key. Disable any signed-in server when using a key-based connection. JSON-RPC `tools/list` returns every tool with its full JSON schema inline. If MCP is unavailable, use REST and read `https://api.moonbounce.ai/openapi.json`. Done when the tool catalog or OpenAPI document is available.
2. List tools and read names **and descriptions**. Find tools by the task descriptions below and read each tool's `inputSchema` before first use; descriptions carry status codes and gotchas. Done when the current tool and schema for each task below are identified.
3. Call the tool described as listing the models this deployment offers; it has no side effects. Done when you can name the offered models and their capabilities.
4. Check the key's environment with `whoami`, which returns the key's org, environment, key id, actor and scopes and spends nothing. Every list page, and every response about per-environment data (evaluations, feedback, cases, previews, suggestions, the model setting and the objective), also carries `environment`; decisions and versions belong to the whole organization, so a decision or version object doesn't. A key is permanently bound to one environment: production, or test with `env:test`. Test traffic never counts as production; its data, previews and model settings are separate. Signing in to production is the normal way for a person to build or improve a decision: scoring uses dry runs, and before `import` the build flow writes only the decision, its model setting and its objective, so don't tell a signed-in person to switch to a test key. A test key, with a fresh decision id such as `skill.<topic>`, is for a separate sandbox: trying the API, or a backend or unattended agent under test. Done when you can state the environment from a response, the offered engines, and the current tools for every task below.

## Tasks

List tools, find the description matching the task, and read its `inputSchema` first. MCP tool names are the snake_case of REST operationIds and SDK methods.

| Task | What to look for in a tool description | Read when doing this task |
| --- | --- | --- |
| Build a decision, with or without a written policy | No single tool: the build flow, below. | [The build flow](references/build.md) |
| Improve a decision in use | No single tool: the improve flow, below. | [The improve flow](references/improve.md) |
| Design a structure | No tool: which facts are attributes, which judgments are questions, which aggregation fits the policy. | [Design a structure](references/design.md) |
| Define a structure | Write JSON in the caller's repo; find “Validate a decision file” (“without registering anything”) to check it publicly, without a key. | [Define a structure](references/tasks.md#define-a-structure) |
| Register it | “Register a decision's version” (“registers its structure as the decision's next version”). | [Register it](references/tasks.md#register-it) |
| Read back or list | “Reads only” descriptions for decisions, their versions and past evaluations. | [Read back or list](references/tasks.md#read-back-or-list) |
| Try without storing | Evaluate with `moonbounce.dry_run: true`: spends model calls, stores only usage. | [Dry run](references/tasks.md#dry-run) |
| Evaluate | “Evaluates one item against a decision”: the structure inline or by hash. | [Evaluate](references/tasks.md#evaluate) |
| Add labeled data | “Imports labeled examples” (`import_examples`) and “Say what the right answer was” (`submit_feedback`). | [Add labeled data](references/tasks.md#add-labeled-data) |
| Backtest a change | “Replays a structure change over past cases with feedback.” | [Backtest a change](references/tasks.md#backtest-a-change) |
| Pick a model | “List the models a decision can run on” (`list_models`), “Get the model a decision runs on” (`get_model`) and “Choose the model a decision runs on” (`set_model`). | [Pick a model](references/tasks.md#pick-a-model) |
| Score a dataset | “Run a version over many items” (`create_run`) and “Judge results under a decision's objective” (`loss_report`). | [Score a dataset](references/tasks.md#score-a-dataset) |
| Tune a cut | “Sweep a cut over a run's answers” (`sweep_cut`): the loss at every point a cut could move to, from a run's answers, with no model calls. | [Tune a cut](references/refine.md#tune-a-cut-with-sweep_cut) |
| List labeled cases | “The decision's labeled cases in the key's environment” (`list_cases`). | [List labeled cases](references/tasks.md#list-labeled-cases) |
| Set the objective | “Set what a decision optimizes for” and “Get what a decision optimizes for” (`set_objective`, `get_objective`). | [Set the objective](references/tasks.md#set-the-objective) |
| Check cost | “Get a decision's model cost, per evaluation and in total” (`get_usage`). | [Check cost](references/tasks.md#check-cost) |
| Wire it into the app | No tool: the app calls `POST /v1/evaluate` with its own API key. | [Wire it into the app](references/tasks.md#wire-it-into-the-app) |

Review-queue tools serve human reviewers; agents rarely need them. These files say how the tools work; the tool descriptions and input schemas are the source of truth, so follow them where the two differ.

## Five rules that bite

1. Register a structure, or evaluate it inline once, before evaluating by its hash; an unknown hash is 404 `STRUCTURE_UNKNOWN`. A dry run is the exception: it also takes the hash of a structure an unexpired run of the decision ran in the same environment, without registering it, and takes a candidate as `structure_changes: {base_hash, changes}`, only the whole features it replaces, adds or removes (null) against that base, merged by the server, whose merged hash the run reports. A backtest does not register its candidate; `register_decision` keeps one by its `structure_hash` alone, from a live run of it.
2. Send the structure one way: inline, as `moonbounce.structure_hash` or (a dry run) as `moonbounce.structure_changes`; two is a 400. Every path a question `reads` (`state.<key>`, `state.<key>.<key>`) must be in `state`, and every attribute without a default needs a value: in `attribute_values`, or at its `from` path in `state` for a derived one, which never goes in `attribute_values` (400 `CONTENT_INVALID`).
3. Read `status` before the result: `decided` carries `outcome`, `computed` carries `value`, `abstained` carries at most the `abstain` outcome.
4. Feedback claims name a question (`expected_answer`) or an outcome the result can produce (`expected_outcome`), or approve/reject the returned outcome (`verdict`, never with `expected_outcome`), or flag broken content (`input_broken`); any other kind is a 400. Previews need a result that decides outcomes (`bands`, `rules` or `match`); a value result is 400 `OUTCOMES_INVALID`. For imported examples, address feedback by the example's evaluation `id` (`about.evaluation_id`); feedback by `item_id` binds to primary traffic only.
5. Send an `idempotency-key` argument on writes except `fill_review_queue`, which takes none. A single-item dry run accepts one and ignores it, so a retried dry run runs, and is billed, again. A run (`create_run`) takes one. Reuse a key only to retry the same request.

## Flows

A person's agent does two things with Moonbounce:

- **The build flow makes a new decision,** from a policy, labeled data or only a description. The brief is what the first structure comes from, and it is the policy from then on.
- **The improve flow improves a decision in use.** The structure exists, and real feedback is the main signal.

Both are lists of named stages, and both end in the same [refine loop](references/refine.md), judged by the decision's objective. Each stage says when it is done, and some stop for the person. A flow uses a team of fresh subagents, so no agent grades its own work.

When the person wants a decision built or improved, by a command (`/moonbounce:start`, `/moonbounce:build` or `/moonbounce:improve` in Claude Code), a prompt or in words, work out the flow and the stage first. Say which you picked and why, then read that flow's guide and follow it from that stage:

1. **The person named a flow**: `/moonbounce:build` or `/moonbounce:improve` in Claude Code, the `build_decision` or `improve_decision` MCP prompt, or in words. Run that flow, resuming at the brief's `stage` when the brief records the same flow. When the build flow is named and the brief records the build flow at `done`, or the improve flow at any stage, ask before starting over: the build flow rewrites the brief. Suggest the improve flow instead. When the improve flow is named and the brief records the build flow unfinished, ask before replacing it; a finished build is where the improve flow starts, at `brief_check`, keeping the brief. An improve flow at `done` starts again at `brief_check`: improving a shipped decision is a new round.
2. **No flow named, and the brief's `stage` isn't `done`:** resume that flow at that stage.
3. **No flow named, and the decision is in use in the key's environment:** it has labeled cases (`list_cases` `total` above 0) or live traffic (`list_evaluations` with `assignment_mode: primary` and `limit: 1` returns one). The improve flow, at `brief_check`.
4. **Otherwise:** the build flow, at `objective`. A decision file already in the repo is the author's first draft.

| Flow | Guide | Stages |
| --- | --- | --- |
| Build | [build.md](references/build.md) | `objective`, `brief`, `cases`, `labels`, `structure`, `refine`, `red_team`, `blind_check`, `import` |
| Improve | [improve.md](references/improve.md) | `brief_check`, `baseline`, `cases`, `refine`, `red_team`, `ship` |

### Where a flow stands

A flow's progress lives in the repo, in the brief's front matter, so a flow resumes from the repo alone. The server keeps nothing about it. The brief is `moonbounce/<decision_id>.brief.md`, beside the decision file `moonbounce/<decision_id>.json`. The build flow's first stage creates it, and the improve flow's first stage drafts one when there is none.

```yaml
---
decision_id: refund-exception
flow: build            # build or improve
stage: labels          # the stage in progress, or done
structure_hash: null   # the structure the brief was last checked against
date: 2026-11-06       # when stage or structure_hash last changed
commits: none          # the person's choice: none, or the branch to commit on after each stage and kept version
frozen: null           # build flow only: a sha256 of the brief's text below the front matter, when the blind set was written
report: null           # chart or text: how the person sees round reports, once known
---
```

Update `stage` and `date` when a stage is done. Going back a stage is allowed: each guide says which stages to re-run after it.

**Committing is the person's call.** When a flow starts, ask once whether they want you to commit as you go (after each stage and each kept version) or to leave committing to them, and if you are to commit, on which branch: suggest `moonbounce/<decision_id>`, and use the one they name. Record it as `commits: <branch>` or `commits: none`. With a branch, create it from the current one at the first commit (or switch to it when it exists), commit only there, and never push. Commit only this decision's files, by path (`git add moonbounce/<decision_id>.* moonbounce/work/<decision_id>/`), never other decisions' files or the person's other changes, and ask before switching branches when they have uncommitted changes outside `moonbounce/`. Without an answer, it is `none`: write the files and leave git alone, branches included. Every file the flow writes stays in the repo either way, so a flow resumes from the files alone.

**Stay inside the repo, one plain command at a time.** The person's agent asks them to approve each command it can't check, and a command that reaches outside the repo, hides its target or carries data inline can't be approved once for good, so it stops the flow every time. So run every command from the folder that holds `moonbounce/`, on relative paths. Every file you or a subagent writes stays under `moonbounce/`, apart from app changes the person approved: scratch scripts and intermediate files go in `moonbounce/scratch/<decision_id>/`, never in `/tmp`; they are never committed and are deleted when the stage is done. Write files with your file tools (Write and Edit in Claude Code), never through the shell: a heredoc (`cat > file <<'EOF'`) or an `echo` of JSON asks every time; a `jq` over named files, redirected to a file under `moonbounce/`, is fine, and so is a `cp` between files under `moonbounce/`. Never redirect onto a file the command reads, since the shell empties it first: write to scratch, then `cp` it over. Run one plain command at a time: no `cd` or `git -C`, no `;` or `&&` chains, loops, shell variables, `$(…)` or `2>/dev/null`; jq's own `$` variables are fine. In jq, write ids and hashes into the filter (`select(.item_id | IN("c-1", "c-2"))`) rather than passing them with `--arg`, `--argjson` or `--slurpfile`. The guides' commands follow this: run them as written, changing only paths, ids and hashes. Never write an API key into a file. Reading this skill's own files is fine. Put this rule in every subagent's prompt.

### What the person sees

Three messages always have the same shape, so the person can compare rounds and knows where to look. Everything else (drafting the brief, explaining a mistake, proposing an attribute) is ordinary conversation.

**Ask through the host's question dialog when it has one** (Claude Code's AskUserQuestion: up to 4 questions a call, which the host shows one at a time; 2 to 4 options a question, plus "Other" for free text). The person picks instead of typing, and **sees one question at a time**, never a batch to answer in one go. Questions that don't depend on each other share a call, up to 4 (a round's probes, the severities, a sample of labels); a question whose answer changes what you'd ask next gets its own call.

- **Each option says what it writes.** The option's description names its effect: a severity's weight ("Costly: weight 3, about three normal mistakes"), what an outcome means, what a target allows.
- **The dialog asks only for the choice.** It has no field for a reason, and "Other" replaces the choice, so never ask for the why inside it.
- **One line of feedback per answer**, after the call and before the next one: what you took from it, what it changed (the weight written, the clause added, the case settled), and the reason you will record, worded as your reading ("I'll note it as 'a change of mind costs us the item'; tell me if that's not why"). When the reading adds or changes a rule in the brief, say so in that line ("this adds a rule: …"), so letting it stand is a real yes. The person sees their answer land and corrects the reason in a few words only when it's wrong. A label the person gives needs its `reason`, and this drafted, uncorrected reading is it.
- **Ask for a reason in text only where you can't guess it:** a label that goes against the brief or against one of their earlier calls, a probe answer that adds a rule, or a costly or unacceptable severity whose why the brief's Mistakes section needs. One short question, right after that answer. At the end of a stage, one question for anything they want to add is fine.
- **What goes in a dialog:**
  - **The objective:** the outcome names first, since the severities name them; then each severity (one question per mistake, a right answer and a wrong one, up to 4 a call), the fallback, the target and commits. The spending cap comes when you give the cost estimate (a few amounts, with "Other").
  - **A probe or a label to settle:** the case in the question's text (the message and the facts that matter), each outcome an option in a fixed order with no recommendation.
  - **A Stop:** the question states in plain words what the stage wrote (the numbers, the choices made), without the word "Stop", and the options are the ways on, your recommendation first and marked (Recommended), plus one to change something first. When that makes more than 4, leave that one out: the dialog's "Other" covers a change.
- **Fall back to text** when the host has no dialog, or when a question has more than 4 outcomes, and then ask one question per message.

**Every way out of a stage that ends in a Stop is the person's choice**, and so is every move back: after `refine`, on to `red_team` or finish; after a red-team round, back to `refine`, another round, on to `blind_check` or finish; after `blind_check`, back to `refine` or on to `import`. Show what the stage found, recommend, and let the person choose, even when the guide names the usual next stage. A stage that ends without a Stop (`cases`, `structure`) moves on when its done-when holds.

**Talk to the person in their words, not this guide's.** "Stop" is the guide's word for a point where you wait for them; never write it to them, in a dialog or in text, including in passing ("I'll bring you the Stop again"). Say what happens instead: "I'll bring you the updated objective to confirm." The same goes for the guide's other terms: say "cost so far", not "the Cost line"; describe a stage the first time you name it ("`red_team`: another model tries to break the decision"). Name a finished stage in plain words: "The objective is set.", "The brief is ready for you to check."

**At a Stop**, use these three parts, in this order:

```markdown
**<The stage, in plain words>: done.**

**What this stage left:** the files written or changed, and the numbers that matter (cases, labels, loss).

**What I need from you:** one question.

**Options:**
1. <the one you recommend> (recommended)
2. <another realistic one>

**Cost:** $0.84 this stage, $4.10 so far of the $20 cap.
```

When the question is how a case should go or which clause wins (a policy ruling, a probe, a label to settle), leave out **Options**: ask what the person would decide (the reason as [the dialog rules](#what-the-person-sees) say: drafted by you, asked for only where you can't guess it), and give your recommendation only once they answer or ask for it. A recommendation shown first tends to be accepted, which hides what they think.

**After each scored candidate in the refine loop**, report the round as a table, base beside candidate, then the candidate's loss matrix, then one line each for the verdict and the change, the decision diff, and the cost:

````markdown
**Round 3: the round 3 candidate against v3**

| | v3 (base) | Round 3 candidate |
| --- | --- | --- |
| Loss per case | 0.32 | 0.17 |
| Macro F1 | 0.84 | 0.91 |
| `<priority outcome>` precision / recall | 0.94 / 0.90 | 0.97 / 0.89 |
| Items scored (`n`) | 118 | 118 |

**Where the candidate's loss is** (rows: actual, columns: predicted)

| actual ↓ predicted → | grant | deny | escalate |
| --- | --- | --- | --- |
| **grant** | ✓ 55 | ▓ 1 (was 2) · 33% | ░ 4 · 7% |
| **deny** | ▓ 3 (was 6) · 47% | ✓ 34 | 1 · 3% |
| **escalate** | 0 (was 1) | ▒ 1 · 10% | ✓ 19 |

**Chart:** <report_url>
**Excluded:** 2 (`failed_transient` 2), the same items on both sides.
**Verdict:** kept as v4 (accepted) / kept as v4 (red-team repair) / dropped: <the report's reasons>; fixed <changes.fixed>, broke <changes.broken>
**Changed:** <the one change> because <the report's evidence>.

**How the decision changed** (v3 → round 3 candidate)

```diff
 questions.late_delivery
 {
   "type": "noul",
-  "instructions": "Is the request inside the return window, counting a late delivery's delay?",
+  "instructions": "Does the tracking line say the delivery was late, or does the customer claim a delay with no tracking line?",
   "reads": ["state.message", "state.order.tracking"]
 }
```

**Cost:** $0.84 this round, $4.10 so far of the $20 cap.
````

**The decision diff** shows the person what the version changed, so the report's question names (in `by_question`, and in the chart's hover card where the host shows it) point at wording they can read. Diff the candidate against the base, by feature: each question, attribute or aggregation added, removed or changed, and the result. Show each changed feature whole, as pretty JSON in a `diff` block, with its changed lines marked (`-` old, `+` new) and each removed line just above the line that replaced it, so the person reads the question as the model sees it. Show an added or removed feature whole too. Leave out a round that changed nothing in the decision (a re-score), and leave out unchanged features: never the whole decision.

The loss matrix is a heatmap of where the loss comes from: the loss the objective's weights put on each kind of wrong answer.

**The chart link:** `loss_report` returns `report_url` when both sides are runs (`run_ids`): a page in the Moonbounce web app that draws the same report as interactive heatmaps. Print it on the **Chart** line, as it came, for every scored version (the first structure's scoring included), so a person reading in a terminal can open it; leave the line out when there is none. It opens for anyone signed in to the team that owns the runs, and expires with them, after about a day.

**Show one report, not two.**

- **A terminal or IDE** (Claude Code, Codex: CLI, desktop or IDE extension) draws no chart inline: show the text matrix (the scorer's table) and the **Chart** line.
- **A chat app that draws MCP Apps** (the Claude apps, ChatGPT): the chart is the report. It draws only for a `loss_report` call made in the main conversation, never a subagent's, so make that call yourself over the scored runs. Show the headline table, the **Verdict**, **Changed** and **Cost** lines, and leave out the text matrix and the decision diff, which the chart shows.
- **When you can't tell,** ask the person once which they see, and record it in the brief's front matter as `report: chart` or `report: text`.

The text matrix:

- **Wrong cells:** the count (from `confusion`), `(was n)` when it differs from the base's `confusion`, and the cell's share of the candidate's loss (its `by_mistake` `loss` ÷ the report's `loss`). Shade by that share: `█` 50% or more, `▓` 25 to 50%, `▒` 10 to 25%, `░` 5 to 10%, nothing below 5%. The shaded cells are where a change pays most.
- **Right cells:** `✓` and the count, from `confusion` (the cells whose `got` is the `expected`).
- **Rows and columns:** the same words as the chart, rows actual and columns predicted, every outcome in the order of the report's `outcomes`, plus a `none` column when some cases got no outcome (`got: null`).
- With more than about six outcomes, list the shaded cells instead of drawing the whole matrix.
- `confusion` lists every cell with a case and `by_mistake` every wrong one, each side's worst or largest 1,000 at most (both sides and both matrices list what either keeps, so a list can hold a few thousand), so a cell missing from them is zero unless `confusion_total` or `by_mistake_total` is larger than the list.

The priority row shows the outcome the objective's `priority` names (with two outcomes), or the outcome whose mistakes cost most. Add a `Target` row only when the person set one, with `target_met`. Show `excluded` only when it isn't empty. In the improve flow, one table per label source that was scored, the deciding one first, named in its title.

**Version numbers are the server's.** A version is vN only once it is kept and registered, N being the `version.number` `register_decision` returned, so reports, commits, `work/runs.json` and the web app agree. Until then a candidate is named by its round ("the round 3 candidate", or "round 3 candidate B" when a round has more than one). A dropped candidate never gets a number.

**Cost** goes in every Stop and every round that ran model calls, in dollars (`cost.usd_micros` ÷ 1,000,000): this run, the running total, and the cap the person set.

For request shapes and task-specific gotchas, read the matching section of [tasks.md](references/tasks.md). Read [install.md](references/install.md) when connecting an agent or installing or sharing this skill.

Conventions:

- Requests are strict; responses may add optional fields, so ignore unknown ones.
- Control enums are closed; error reasons and model identifiers are open. Read `error.details.issues` as `{param, code, message}`, and fall back to the status and error type for a code you don't know.
- On 403 `ORG_SUSPENDED` (`details.cause` is `unpaid` or `operator`) or `ORG_CLOSED`, stop and tell the person their team needs billing or support attention; don't retry.
- Names: `structure` is the object, `structure_hash` and `question_hashes` its hashes, `decision_id` and `item_id` the caller's ids, `review_item_id` a review item (`inb_`). Counts are `claim_count`, claim arrays `claims`.
- Paginated lists end with `next_cursor: null` and `has_more: false`.
- Stored inputs read as `content: {state, attribute_values?}`, or `content: null` when not stored. `get_evaluation`'s `attribute_values` are resolved (defaults and derived values included, the derived ones named in `derived_attributes`), so don't send them back as a call's `attribute_values`; `list_cases` content holds only what the caller sent.
