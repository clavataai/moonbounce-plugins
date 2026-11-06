---
name: decisions
description: "Use when calling the Moonbounce Decisions MCP or API: build a decision from a policy (flow 1) or improve one in use (flow 2), resuming where the brief says; define/register/evaluate a decision, read back or list decisions, structures, evaluations and labeled cases, import labeled examples or send feedback, score a dataset, backtest, set the objective, pick an engine, check cost."
---

# Moonbounce Decisions

A decision is a JSON **structure** in the System One shape: `questions` an engine answers over the call's `state` (each may list the `state.<key>` paths it `reads`, nested ones like `state.order.total` included), optional `attributes` the caller supplies, `aggregations` that combine them (weights, thresholds and cut points written in), and a `result` naming the final feature. Every reference inside a structure is a path: `questions.<name>`, `attributes.<name>` or `aggregations.<name>` (`"result": "aggregations.handling"`); a bare name is a 400. The caller's code owns the structure; Moonbounce runs it, records evaluations and learns from feedback. Imported examples, feedback, review and previews all take this shape.

## Discovery routine — first, every session

1. Use the connected `moonbounce` MCP server (with the Claude Code plugin it appears as `plugin:moonbounce:moonbounce`). If it's listed but its tools aren't available because it needs sign-in, you can't start sign-in yourself: ask the person to run `/mcp` → the server → **Authenticate** (or `claude mcp login <name>` in a shell, for a server they added by hand), then continue. If none, connect your agent to `https://api.moonbounce.ai/mcp` by sign-in or key; see [per-agent installation](references/install.md). Sign-in gives admins and members `decisions:develop` and `decisions:evaluate` in Production, including real and dry-run evaluations and reads, attributed to the person. Unattended agents, backends, Test (`env:test`) and REST need an API key with `decisions:develop`/`decisions:evaluate` as needed; send it as `Authorization: Bearer <key>`. Never print the key. Disable any signed-in server when using a key-based connection. JSON-RPC `tools/list` returns every tool with its full JSON schema inline (observed 2026-10-27). If MCP is unavailable, use REST and read `https://api.moonbounce.ai/openapi.json` (observed 2026-10-27 to match the reference API schema). Done when the tool catalog or OpenAPI document is available.
2. List tools and read names **and descriptions**. Find tools by the task descriptions below and read each tool's `inputSchema` before first use; descriptions carry status codes and gotchas. Done when the current tool and schema for each task below are identified.
3. Call the tool described as listing the engines this deployment offers; it has no side effects. Done when you can name the offered engines and their capabilities.
4. Check the key's environment with `whoami`, which returns the key's org, environment, key id, actor and scopes and spends nothing. Every response that reads or writes your data also carries `environment`. A key is permanently bound to one environment: production, or test with `env:test`. Test traffic never counts as production; its data, previews and bindings are separate. Use a test key while learning and a fresh decision id such as `skill.<topic>`. Done when you can state the environment from a response, the offered engines, and the current tools for every task below.

## Tasks

List tools, find the description matching the task, and read its `inputSchema` first. MCP tool names are the snake_case of REST operationIds and SDK methods.

| Task | What to look for in a tool description | Read when doing this task |
| --- | --- | --- |
| Build a decision from a policy | No single tool: flow 1, below. | [Flow 1: build](references/build.md) |
| Improve a decision in use | No single tool: flow 2, below. | [Flow 2: improve](references/improve.md) |
| Define a structure | Write JSON in the caller's repo; find “Validate and hash a decision structure” (“without registering it”) to check it publicly, without a key. | [Define a structure](references/tasks.md#define-a-structure) |
| Register it | “Stores a structure under the decision so evaluations can name it in `moonbounce.structure_hash`.” | [Register it](references/tasks.md#register-it) |
| Read back or list | “Reads only” descriptions for decisions, registered structures and past evaluations. | [Read back or list](references/tasks.md#read-back-or-list) |
| Try without storing | Evaluate with `moonbounce.dry_run: true`: spends engine calls, stores only usage. | [Dry run](references/tasks.md#dry-run) |
| Evaluate | “Evaluates one item against a decision”: the structure inline or by hash. | [Evaluate](references/tasks.md#evaluate) |
| Add labeled data | “Imports labeled examples” (`import_examples`) and “Say what the right answer was” (`submit_feedback`). | [Add labeled data](references/tasks.md#add-labeled-data) |
| Backtest a change | “Replays a structure change over past cases with feedback.” | [Backtest a change](references/tasks.md#backtest-a-change) |
| Pick an engine | “The engines this deployment offers,” “The decision's current binding revision,” and “Sets the engine and model a decision runs on.” | [Pick an engine](references/tasks.md#pick-an-engine) |
| Score a dataset | `evaluate` with `items` (“**A run**” in its description) and “Judges one version's labeled results” (`loss_report`). | [Score a dataset](references/tasks.md#score-a-dataset) |
| List labeled cases | “The decision's labeled cases in the key's environment” (`list_cases`). | [List labeled cases](references/tasks.md#list-labeled-cases) |
| Set the objective | “Stores the decision file's `objective` as a new revision” and “The decision's current objective revision” (`set_objective`, `get_objective`). | [Set the objective](references/tasks.md#set-the-objective) |
| Check cost | “Engine cost of one decision's evaluations and previews over a window.” | [Check cost](references/tasks.md#check-cost) |

Review-queue tools serve human reviewers; agents rarely need them. Statements here without a date are from the tool descriptions and Moonbounce's guides; the references tag each claim **observed** (run live, test key, 2026-10-27) or **from docs**.

## Five rules that bite

1. Register a structure, or evaluate it inline once, before evaluating by its hash; an unknown hash is 404 `STRUCTURE_UNKNOWN`. A backtest does not register its candidate.
2. Send the structure inline or as `moonbounce.structure_hash`, never both (400). Every path a question `reads` (`state.<key>`, `state.<key>.<key>`) must be in `state`, and every attribute without a default needs a value in `attribute_values` (400 `CONTENT_INVALID`).
3. Read `status` before the result: `decided` carries `outcome`, `computed` carries `value`, `abstained` carries at most the `abstain` outcome.
4. Feedback claims name a question (`expected_answer`) or an outcome the result can produce (`expected_outcome`), or approve/reject the returned outcome (`verdict`, never with `expected_outcome`), or flag broken content (`input_broken`); any other kind is a 400. Previews need a result that decides outcomes (`bands`, `rules` or `match`); a value result is 400 `OUTCOMES_INVALID`. For imported examples, address feedback by the example's evaluation `id` (`about.evaluation_id`); feedback by `item_id` binds to primary traffic only.
5. Send an `idempotency-key` argument on writes except dry runs, which refuse it, and `fill_review_queue`, which takes none. Reuse a key only to retry the same request.

## Flows

A person's agent does two things with Moonbounce:

- **Flow 1 builds a decision from a policy.** The brief is what the first structure comes from.
- **Flow 2 improves a decision in use.** The structure exists, and real feedback is the main signal.

Both are lists of named stages, and both end in the same [refine loop](references/refine.md), judged by the decision's objective. Each stage says when it is done, and some stop for the person. A flow uses a team of fresh subagents, so no agent grades its own work.

When the person wants a decision built or improved, by a command (`/moonbounce:start`, `/moonbounce:build` or `/moonbounce:improve` in Claude Code), a prompt or in words, work out the flow and the stage first. Say which you picked and why, then read that flow's guide and follow it from that stage:

1. **The person named a flow**: `/moonbounce:build` or `/moonbounce:improve` in Claude Code, the `build_decision` or `improve_decision` MCP prompt, or in words. Run that flow, resuming at the brief's `stage` when the brief records the same flow. If the brief records the other flow, unfinished, ask before replacing it.
2. **No flow named, and the brief's `stage` isn't `done`:** resume that flow at that stage.
3. **No flow named, and the decision has labeled cases in the key's environment** (`list_cases` `total` above 0): flow 2, at `brief_check`.
4. **Otherwise:** flow 1, at `objective`. A decision file already in the repo is the author's first draft.

| Flow | Guide | Stages |
| --- | --- | --- |
| 1, build | [build.md](references/build.md) | `objective`, `brief`, `cases`, `labels`, `structure`, `refine`, `blind_check`, `import` |
| 2, improve | [improve.md](references/improve.md) | `brief_check`, `baseline`, `cases`, `refine`, `ship` |

### Where a flow stands

A flow's progress lives in the repo, in the brief's front matter, so a flow resumes from the repo alone. The server keeps nothing about it. The brief is `moonbounce/<decision_id>.brief.md`, beside the decision file `moonbounce/<decision_id>.json`. Flow 1's first stage creates it, and flow 2's first stage drafts one when there is none.

```yaml
---
decision_id: refund-exception
flow: build            # build (flow 1) or improve (flow 2)
stage: labels          # the stage in progress, or done
structure_hash: null   # the structure the brief was last checked against
date: 2026-11-06       # when stage or structure_hash last changed
frozen: null           # flow 1: the commit of the brief the blind set was written from
---
```

Update `stage` and `date` when a stage is done, and commit. Going back a stage is allowed: each guide says which stages to re-run after it.

For request shapes and task-specific gotchas, read the matching section of [tasks.md](references/tasks.md). For evidence behind these rules, docs-versus-live differences, unverified behavior, or changes that may invalidate them, read [verified.md](references/verified.md). Read [install.md](references/install.md) when connecting an agent or installing or sharing this skill.

Requests are strict. Responses may add optional fields; ignore unknown fields. Control enums are closed, while error reasons and engine identifiers are open catalogs. Read `error.details.issues` as `{param,code,message}`; fall back to status and error type for an unknown code. On 403 `ORG_SUSPENDED` (`details.cause` is `unpaid` or `operator`) or `ORG_CLOSED`, stop and tell the user their team needs billing or support attention; don't retry. Public names are `structure` for the object, `structure_hash` and `question_hashes` for hashes, `decision_id`/`item_id` for caller IDs, and `review_item_id` for review route arguments (`inb_` IDs). Counts are `claim_count`, claim arrays are `claims`. Paginated lists end with `next_cursor:null` and `has_more:false`. Read stored inputs in `content:{state,attribute_values?}`, or `content:null` when not stored.
