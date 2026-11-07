---
name: decisions
description: "Use when calling the Moonbounce Decisions MCP or API: build a decision from a policy (the build flow) or improve one in use (the improve flow), resuming where the brief says; define/register/evaluate a decision, read back or list decisions, versions, evaluations and labeled cases, import labeled examples or send feedback, score a dataset, backtest, set the objective, pick an engine, check cost."
---

# Moonbounce Decisions

A decision is a JSON **structure**: `questions` an engine answers over the call's `state` (each may list the `state.<key>` paths it `reads`, nested ones like `state.order.total` included), optional `attributes` the caller supplies, `aggregations` that combine them (weights, thresholds and cut points written in), and a `result` naming the final feature. Every reference inside a structure is a path: `questions.<name>`, `attributes.<name>` or `aggregations.<name>` (`"result": "aggregations.handling"`); a bare name is a 400. The caller's code owns the structure; Moonbounce runs it, records evaluations and learns from feedback. Imported examples, feedback, review and previews all take this shape.

## Discovery routine — first, every session

1. Use the connected `moonbounce` MCP server (with the Claude Code plugin it appears as `plugin:moonbounce:moonbounce`). If it's listed but its tools aren't available because it needs sign-in, you can't start sign-in yourself: ask the person to run `/mcp` → the server → **Authenticate** (or `claude mcp login <name>` in a shell, for a server they added by hand), then continue. If none, connect your agent to `https://api.moonbounce.ai/mcp` by sign-in or key; see [per-agent installation](references/install.md). Sign-in gives admins and members `decisions:develop` and `decisions:evaluate` in Production, including real and dry-run evaluations and reads, attributed to the person. Unattended agents, backends, Test (`env:test`) and REST need an API key with `decisions:develop`/`decisions:evaluate` as needed; send it as `Authorization: Bearer <key>`. Never print the key. Disable any signed-in server when using a key-based connection. JSON-RPC `tools/list` returns every tool with its full JSON schema inline. If MCP is unavailable, use REST and read `https://api.moonbounce.ai/openapi.json`. Done when the tool catalog or OpenAPI document is available.
2. List tools and read names **and descriptions**. Find tools by the task descriptions below and read each tool's `inputSchema` before first use; descriptions carry status codes and gotchas. Done when the current tool and schema for each task below are identified.
3. Call the tool described as listing the engines this deployment offers; it has no side effects. Done when you can name the offered engines and their capabilities.
4. Check the key's environment with `whoami`, which returns the key's org, environment, key id, actor and scopes and spends nothing. Every list page, and every response about per-environment data (evaluations, feedback, cases, previews, suggestions, the binding and the objective), also carries `environment`; decisions and versions belong to the whole organization, so a decision or version object doesn't. A key is permanently bound to one environment: production, or test with `env:test`. Test traffic never counts as production; its data, previews and bindings are separate. Signing in to production is the normal way for a person to build or improve a decision: scoring uses dry runs, and before `import` the build flow writes only the decision, its binding and its objective, so don't tell a signed-in person to switch to a test key. A test key, with a fresh decision id such as `skill.<topic>`, is for a separate sandbox: trying the API, or a backend or unattended agent under test. Done when you can state the environment from a response, the offered engines, and the current tools for every task below.

## Tasks

List tools, find the description matching the task, and read its `inputSchema` first. MCP tool names are the snake_case of REST operationIds and SDK methods.

| Task | What to look for in a tool description | Read when doing this task |
| --- | --- | --- |
| Build a decision from a policy | No single tool: the build flow, below. | [The build flow](references/build.md) |
| Improve a decision in use | No single tool: the improve flow, below. | [The improve flow](references/improve.md) |
| Design a structure | No tool: which facts are attributes, which judgments are questions, which aggregation fits the policy. | [Design a structure](references/design.md) |
| Define a structure | Write JSON in the caller's repo; find “Validate a decision file” (“without registering anything”) to check it publicly, without a key. | [Define a structure](references/tasks.md#define-a-structure) |
| Register it | “Register a decision's version” (“registers its structure as the decision's next version”). | [Register it](references/tasks.md#register-it) |
| Read back or list | “Reads only” descriptions for decisions, their versions and past evaluations. | [Read back or list](references/tasks.md#read-back-or-list) |
| Try without storing | Evaluate with `moonbounce.dry_run: true`: spends engine calls, stores only usage. | [Dry run](references/tasks.md#dry-run) |
| Evaluate | “Evaluates one item against a decision”: the structure inline or by hash. | [Evaluate](references/tasks.md#evaluate) |
| Add labeled data | “Imports labeled examples” (`import_examples`) and “Say what the right answer was” (`submit_feedback`). | [Add labeled data](references/tasks.md#add-labeled-data) |
| Backtest a change | “Replays a structure change over past cases with feedback.” | [Backtest a change](references/tasks.md#backtest-a-change) |
| Pick an engine | “The engines this deployment offers,” “The decision's current binding revision,” and “Sets the engine and model a decision runs on.” | [Pick an engine](references/tasks.md#pick-an-engine) |
| Score a dataset | `evaluate` with `items` (“**A run**” in its description) and “Judges one version's labeled results” (`loss_report`). | [Score a dataset](references/tasks.md#score-a-dataset) |
| Tune a cut | “Sweep a cut over a run's answers” (`sweep_cut`): the loss at every point a cut could move to, from a run's answers, with no engine calls. | [Tune a cut](references/refine.md#tune-a-cut-with-sweep_cut) |
| List labeled cases | “The decision's labeled cases in the key's environment” (`list_cases`). | [List labeled cases](references/tasks.md#list-labeled-cases) |
| Set the objective | “Stores the decision file's `objective` as a new revision” and “The decision's current objective revision” (`set_objective`, `get_objective`). | [Set the objective](references/tasks.md#set-the-objective) |
| Check cost | “Engine cost of one decision's evaluations and previews over a window.” | [Check cost](references/tasks.md#check-cost) |
| Wire it into the app | No tool: the app calls `POST /v1/evaluate` with its own API key. | [Wire it into the app](references/tasks.md#wire-it-into-the-app) |

Review-queue tools serve human reviewers; agents rarely need them. These files say how the tools work; the tool descriptions and input schemas are the source of truth, so follow them where the two differ.

## Five rules that bite

1. Register a structure, or evaluate it inline once, before evaluating by its hash; an unknown hash is 404 `STRUCTURE_UNKNOWN`. A backtest does not register its candidate.
2. Send the structure inline or as `moonbounce.structure_hash`, never both (400). Every path a question `reads` (`state.<key>`, `state.<key>.<key>`) must be in `state`, and every attribute without a default needs a value in `attribute_values` (400 `CONTENT_INVALID`).
3. Read `status` before the result: `decided` carries `outcome`, `computed` carries `value`, `abstained` carries at most the `abstain` outcome.
4. Feedback claims name a question (`expected_answer`) or an outcome the result can produce (`expected_outcome`), or approve/reject the returned outcome (`verdict`, never with `expected_outcome`), or flag broken content (`input_broken`); any other kind is a 400. Previews need a result that decides outcomes (`bands`, `rules` or `match`); a value result is 400 `OUTCOMES_INVALID`. For imported examples, address feedback by the example's evaluation `id` (`about.evaluation_id`); feedback by `item_id` binds to primary traffic only.
5. Send an `idempotency-key` argument on writes except a single-item dry run, which refuses it, and `fill_review_queue`, which takes none. A run (`items` or `cases`) takes one. Reuse a key only to retry the same request.

## Flows

A person's agent does two things with Moonbounce:

- **The build flow makes a new decision from a policy.** The brief is what the first structure comes from.
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
---
```

Update `stage` and `date` when a stage is done. Going back a stage is allowed: each guide says which stages to re-run after it.

**Committing is the person's call.** When a flow starts, ask once whether they want you to commit as you go (after each stage and each kept version) or to leave committing to them, and if you are to commit, on which branch: suggest `moonbounce/<decision_id>`, and use the one they name. Record it as `commits: <branch>` or `commits: none`. With a branch, create it from the current one at the first commit (or switch to it when it exists), commit only there, and never push. Commit only this decision's files, by path (`git add moonbounce/<decision_id>.* moonbounce/work/<decision_id>/`), never other decisions' files or the person's other changes, and ask before switching branches when they have uncommitted changes outside `moonbounce/`. Without an answer, it is `none`: write the files and leave git alone, branches included. Every file the flow writes stays in the repo either way, so a flow resumes from the files alone.

**Stay inside the repo.** The person's agent asks them to approve each command it can't check, and a command that reaches outside the repo or hides its target can't be approved once for good. So run every command from the folder that holds `moonbounce/`, on relative paths. Every file you or a subagent writes stays under `moonbounce/`, apart from app changes the person approved: scratch scripts and intermediate files go in `moonbounce/scratch/<decision_id>/`, never in `/tmp`; they are never committed and are deleted when the stage is done. Don't wrap commands in `cd … &&` or use `git -C`, and don't assign shell variables or use `$(…)` in a command; jq's own `$` variables are fine. Never write an API key into a file. Reading this skill's own files is fine. Put this rule in every subagent's prompt.

### What the person sees

Three messages always have the same shape, so the person can compare rounds and knows where to look. Everything else (drafting the brief, explaining a mistake, proposing an attribute) is ordinary conversation.

**At a Stop**, use these three parts, in this order:

```markdown
**Stop: `<stage>` is done.**

**What this stage left:** the files written or changed, and the numbers that matter (cases, labels, loss).

**What I need from you:** one question.

**Options:**
1. <the one you recommend> (recommended)
2. <another realistic one>

**Cost:** $0.84 this stage, $4.10 so far of the $20 cap.
```

**After each scored candidate in the refine loop**, report the round as a table, base beside candidate, then the candidate's loss matrix, then one line each for the verdict, the change and the cost:

```markdown
**Round 3: v4 against v3**

| | v3 (base) | v4 (candidate) |
| --- | --- | --- |
| Loss per case | 0.32 | 0.17 |
| Macro F1 | 0.84 | 0.91 |
| `<priority outcome>` precision / recall | 0.94 / 0.90 | 0.97 / 0.89 |
| Items scored (`n`) | 118 | 118 |

**Where v4's loss is** (row: the right answer; column: what v4 gave)

| right ↓ gave → | grant | deny | escalate |
| --- | --- | --- | --- |
| **grant** | ✓ 55 | ▓ 1 (was 2) · 33% | ░ 4 · 7% |
| **deny** | ▓ 3 (was 6) · 47% | ✓ 34 | 1 · 3% |
| **escalate** | 0 (was 1) | ▒ 1 · 10% | ✓ 19 |

**Excluded:** 2 (`failed_transient` 2), the same items on both sides.
**Verdict:** kept (accepted) / kept (red-team repair) / dropped: <the report's reasons>
**Changed:** <the one change> because <the report's evidence>.
**Cost:** $0.84 this round, $4.10 so far of the $20 cap.
```

The loss matrix is a heatmap of where the loss comes from: the loss the objective's weights put on each kind of wrong answer.

- **Wrong cells:** the count (from `confusion`), `(was n)` when it differs from the base's `confusion`, and the cell's share of the candidate's loss (its `by_mistake` `loss` ÷ the report's `loss`). Shade by that share: `█` 50% or more, `▓` 25 to 50%, `▒` 10 to 25%, `░` 5 to 10%, nothing below 5%. The shaded cells are where a change pays most.
- **Right cells:** `✓` and the count, from `confusion` (the cells whose `got` is the `expected`).
- **Columns:** every outcome, plus `none` when some cases got no outcome (`got: null`).
- With more than about six outcomes, list the shaded cells instead of drawing the whole matrix.

The priority row shows the outcome the objective's `priority` names (with two outcomes), or the outcome whose mistakes cost most. Add a `Target` row only when the person set one, with `target_met`. Show `excluded` only when it isn't empty. In the improve flow, one table per label source that was scored, the deciding one first, named in its title. Keep the version numbers (v1, v2, …) the same in every report, the decision file's commits and `work/runs.json`.

**Cost** goes in every Stop and every round that ran engine calls, in dollars (`cost.usd_micros` ÷ 1,000,000): this run, the running total, and the cap the person set.

For request shapes and task-specific gotchas, read the matching section of [tasks.md](references/tasks.md). Read [install.md](references/install.md) when connecting an agent or installing or sharing this skill.

Conventions:

- Requests are strict; responses may add optional fields, so ignore unknown ones.
- Control enums are closed; error reasons and engine identifiers are open. Read `error.details.issues` as `{param, code, message}`, and fall back to the status and error type for a code you don't know.
- On 403 `ORG_SUSPENDED` (`details.cause` is `unpaid` or `operator`) or `ORG_CLOSED`, stop and tell the person their team needs billing or support attention; don't retry.
- Names: `structure` is the object, `structure_hash` and `question_hashes` its hashes, `decision_id` and `item_id` the caller's ids, `review_item_id` a review item (`inb_`). Counts are `claim_count`, claim arrays `claims`.
- Paginated lists end with `next_cursor: null` and `has_more: false`.
- Stored inputs read as `content: {state, attribute_values?}`, or `content: null` when not stored.
