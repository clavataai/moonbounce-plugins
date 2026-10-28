---
name: decisions
description: "Use when calling the Moonbounce Decisions MCP or API: define/register/evaluate a decision, read back or list decisions, structures and evaluations, seed or send feedback, backtest, suggestions, pick an engine, check cost."
---

# Moonbounce Decisions

A decision is a JSON spec of named `nodes`: boolean features answered by an engine, `combine`/`calibrate` aggregators, an `output`, and a `target`. The caller's code owns the spec and the cut points in `outcomes`; Moonbounce records evaluations and learns from feedback.

## Discovery routine — first, every session

1. Use the connected `moonbounce` MCP server (with the Claude Code plugin it appears as `plugin:moonbounce:moonbounce`). If none, connect your agent to `https://app.moonbounce.ai/mcp` by sign-in or key; see [per-agent installation](references/install.md). Sign-in gives admins and members `decisions:develop` and `decisions:evaluate` in Production, including real and dry-run evaluations and reads, attributed to the person. Unattended agents, backends, Test (`env:test`) and REST need an API key with `decisions:develop`/`decisions:evaluate` as needed; send it as `Authorization: Bearer <key>`. Never print the key. Disable any signed-in server when using a key-based connection. JSON-RPC `tools/list` returns every tool with its full JSON schema inline (observed 2026-10-27). If MCP is unavailable, use REST and read `https://app.moonbounce.ai/openapi.json` (observed 2026-10-27 to match the reference API schema). Done when the tool catalog or OpenAPI document is available.
2. List tools and read names **and descriptions**. Find tools by the task descriptions below and read each tool's `inputSchema` before first use; descriptions carry status codes and gotchas. Done when the current tool and schema for each task below are identified.
3. Call the tool described as listing the engines this deployment offers; it has no side effects. Done when you can name the offered engines and their capabilities.
4. Check the key's environment with `whoami`, which returns the key's org, environment, key id, actor and scopes and spends nothing. Every response that reads or writes your data also carries `environment`. A key is permanently bound to one environment: production, or test with `env:test`. Test traffic never counts as production; its data, previews, suggestions, and bindings are separate. Use a test key while learning and a fresh decision id such as `skill.<topic>`. Done when you can state the environment from a response, the offered engines, and the current tools for every task below.

## Tasks

List tools, find the description matching the task, and read its `inputSchema` first. MCP tool names are the snake_case of REST operationIds and SDK methods.

| Task | What to look for in a tool description | Read when doing this task |
| --- | --- | --- |
| Define a spec | Write JSON in the caller's repo; find “Hash a decision spec ... without registering it” to check it publicly, without a key. | [Define a spec](references/tasks.md#define-a-spec) |
| Register it | “Stores a spec's structure under the decision so evaluations can name it by hash.” | [Register it](references/tasks.md#register-it) |
| Read back or list | “Reads only” descriptions for decisions, registered structures and past evaluations. | [Read back or list](references/tasks.md#read-back-or-list) |
| Try without storing | Evaluate with `dry_run: true`: spends engine calls, stores only usage. | [Dry run](references/tasks.md#dry-run) |
| Evaluate | “Scores one item with a spec (inline, or by registered hash).” | [Evaluate](references/tasks.md#evaluate) |
| Add labeled data | “Imports labeled starting examples (seeds)” and “Say what the right answer was.” | [Add labeled data](references/tasks.md#add-labeled-data) |
| Backtest a change | “Replays a spec change over past cases with feedback.” | [Backtest a change](references/tasks.md#backtest-a-change) |
| Ask for suggestions | “Proposes one change to the decision's spec from its feedback”; also find list/get/status and “Apply a Moonbounce suggestion to this repo.” | [Ask for suggestions](references/tasks.md#ask-for-suggestions) |
| Pick an engine | “The engines this deployment offers,” “The decision's current binding revision,” and “Sets the engine and model a decision runs on.” | [Pick an engine](references/tasks.md#pick-an-engine) |
| Check cost | “Engine cost of one decision's evaluations and previews over a window.” | [Check cost](references/tasks.md#check-cost) |

Review-queue tools serve human reviewers; agents rarely need them. Statements here without a date are from the tool descriptions and Moonbounce's guides; the references tag each claim **observed** (run live, test key, 2026-10-27) or **from docs**.

## Five rules that bite

1. Register a backtested candidate, or evaluate it inline once, before evaluating by its structure hash. Backtesting currently leaves the candidate unregistered.
2. Address feedback for seeded items with the `evaluation_id` from the seed response; item-id lookup binds to primary evaluations and fails on seed-only items.
3. Keep cut-point tuning separate from today's backtest: it burns the held-out cases it lists and exposes only changed items, not the score distribution.
4. Account for mean-prior dilution when adding features; `prior: "any"` can saturate near 1.0. Match fixed-weight signs to each term's effect; the server refuses mismatches with 400.
5. Send an `idempotency-key` argument on writes except dry runs, which refuse it. Reuse a key only to retry the same request.

For request shapes and task-specific gotchas, read the matching section of [tasks.md](references/tasks.md). For evidence behind these rules, docs-versus-live differences, unverified behavior, or changes that may invalidate them, read [verified.md](references/verified.md). Read [install.md](references/install.md) when connecting an agent or installing or sharing this skill.
