# Tasks

Evidence convention: **observed** means run live against the production host with a test-environment key on 2026-10-27, before the System One shape (an observation is kept only where it still applies in substance); **from docs** means read only (Moonbounce's guides, tool descriptions or the contract); **from code, reproduced locally** means read in the source and run against a local database. The System One request shape (structures, evaluate, read-backs) has not been checked live yet, so those sections are **from docs**. Request fields and response shapes below use the current naming; use the discovered schema for each request.

Before each task, list tools, find the one whose description matches the goal, and read its `inputSchema` before first use. For REST, match the operation in the current `/openapi.json` and read its schema.

## Define a structure

**Goal:** Create the decision JSON in the caller's repo and obtain its structure hash.

**Find:** **from docs** — No tool writes the structure. Find “Validate and hash a decision structure” (`hash_structure`, `POST /v1/structures/hash`); this public check needs no key and writes nothing. Locally, `mb structure hash <file>` hashes a bare structure file (`--local` skips the API). Read the `inputSchema` first.

**Shape:** **from docs** — The body is the structure itself:

- `questions`: `{type:"noul", instructions, criteria?:{true?,false?}, reads?:["state.<key>", "state.<key>.<key>", …]}` (object paths at any depth, sent nested as in the state; array indexes such as `state.items[0]` are a 400, and so is an entry repeated or containing another, like `state.order` with `state.order.id`). `noul` is the only kind today; its answer is P(true). `instructions` is a string, object or array, sent as written. Without `reads` a question sees the whole state.
- `attributes` (optional): values the caller sends per call in `attribute_values`, never shown to the engine: `{type:"string", enum, default?, source?}`, `{type:"number", minimum?, maximum?, default?, source?}` or `{type:"boolean", default?, source?}`.
- `aggregations` (optional): `weighted_mean` (positive weights or none, `invert`), `logistic` (`bias`, signed `weight` on every input), `min`/`max`/`product`/`noisy_or` (`invert`), checks `all`/`any` (`gte`/`gt`/`lte`/`lt`, default ≥ 0.5), and outcome types `bands` (`input`, `below`, ascending `cuts:[{at,outcome}]`, `abstain?`), `rules` (`rules:[{when,outcome}]`, `else`) and `match` (`input`, `cases`, `else?`, `abstain?`). Numeric and check types list `inputs:[{name,...}]`; `bands`/`match` read `input`, and `rules` read their conditions. Every reference (`inputs[].name`, `input`, a rule's `name`, `result`) is a path: `questions.<name>`, `attributes.<name>` or `aggregations.<name>`.
- `result`: the final feature's path (`questions.x`, `attributes.x` or `aggregations.x`).

**Gotchas:** **from docs** — Names are lowercase slugs, unique across questions, attributes and aggregations. A bare name, a path under the wrong root or an unknown name where a reference goes is a 400 naming the path; the map keys, `answers`, `aggregations` and claims stay bare. Hashing ignores key order, NFC normalization of instructions, criteria and attribute sources, the order of `reads` and `inputs`, and spelled-out defaults (`invert:false`, `gte:0.5`, empty `criteria`, all-1 `weighted_mean` weights). Read paths, string enum labels/defaults and label comparisons stay exact, since customer keys and values are not normalized. Any other edit, including a weight, threshold or cut point, is a new hash. Each question also gets its own hash (its definition without its name). Cycles between aggregations are a 400 that names the cycle. There is no weights hash.

**Done when:** The JSON is in the caller's repo and the checking tool returns its structure hash.

## Register it

**Goal:** Make a structure available to evaluations by hash in the chosen environment.

**Find:** **from docs** — Find “Stores a structure under the decision so evaluations can name it in `moonbounce.structure_hash`” (`register_structure`, `POST /v1/decisions/{decision_id}/structures`). Read its `inputSchema` first.

**Request:** **from docs** — The body is the structure itself, with the decision id in the path. 201 when this call registered it, 200 when it already existed; the response carries `structure_hash` and per-question hashes in `question_hashes`, not the structure object. A normal evaluation with an inline structure also registers it on first sight; a dry run does not.

**Gotchas:** **observed** — First registration in an environment creates the engine binding; the checked deployment selected Jev at revision 1. A backtest does not register its candidate.

**Done when:** Registration succeeds and the structure hash is available for that decision in the intended environment.

## Evaluate

**Goal:** Run one item through the caller's structure.

**Find:** **from docs** — Find “Evaluates one item against a decision” (`evaluate`). Read its `inputSchema` first; its description ends with a worked example `body`.

**Request fields:** **from docs** — `state` (any JSON; an object when questions have `reads`); the structure inline (`questions`, `attributes?`, `aggregations?`, `result`) or its hash in `moonbounce.structure_hash`, never both; `attribute_values` when attributes are declared; optional `model` (must match the binding); and `moonbounce: {decision_id, item_id, structure_hash?, metadata?, summary?, dry_run?}`. A plain System One body (`model`, `state`, `questions`) plus `result` and `moonbounce` is valid.

**Result:** **from docs** — `id`, `status`, `value`, `outcome`, `via`, `missing` (the result's questions with no answer, as `questions.x`; absent when none, so gate on `status === "decided" && !missing` before acting on full evidence), `answers` (each `{type:"noul", noul}` or `abstained`/`failed`), `aggregations` (each `value`, or `outcome`/`via`, `missing` (input paths), `abstained`), `moonbounce` with the resolved `structure_hash`, `version: {structure_hash, binding, engine_profile}`, per-question `engines`, `usage`, `cost` and `calls` (one per upstream System One call: `questions`, the `reads` paths it sent, `input_tokens`).

**Gotchas:** **from docs** — Questions that read the same paths share one engine call; a key nobody reads is never sent. An unknown hash is 404 `STRUCTURE_UNKNOWN`; a missing read path (any level, or a non-object on the way) or a bad attribute value is 400 `CONTENT_INVALID` listing every issue. A question on the result's path that fails is 502 `ENGINE_FAILED` (503 when transient); one off the path reports `failed`. **observed** — The budget is 10 seconds; the `prefer` argument can shorten it. **from docs** — The response carries `environment`; check it says the environment you meant. Preserve the evaluation `id`.

**Cost:** **from docs** — Fresh responses, including dry runs, carry `cost: {engine_calls, cache_hits, input_tokens, usd_micros}` with the same fields as a preview. `engine_calls` counts upstream engine calls; `calls` shows which questions and keys each one carried. Evaluate reports `cache_hits: 0`. Keep `usage` for input and output token counts.

**Status:** **from docs** — `decided` (an outcome aggregation named `outcome`; `bands` adds the `value` it cut, `rules`/`match` add `via`), `computed` (the result is a number in `value`) or `abstained` (no answer; `outcome` is the `abstain` outcome when one is set). `via` is a zero-based rule index, `cases.<exact label>` for a matched case (so a case named `else` is `cases.else`), or `else` for the fallback. Check `status` before reading the rest.

**Done when:** A response identifies the evaluation, its `status`, the version used and the outcome or value.

## Add labeled data

**Goal:** Supply starting examples or judgments about existing evaluations, then inspect agreement and conflicts.

**Shape:** **from docs** — Seeds and feedback take the System One shape (not yet rechecked live). Claims work at two levels: `expected_answer` `{question, value: true|false}` for one question, and `expected_outcome` `{value}` (one of the outcome names the result can produce) or `verdict` `{value: "approve"|"reject"}` for the decision; plus `input_broken` `{problem}`. The previous claim kinds (`outcome`, `target`, `feature`, `action`) are refused; update them before sending. A verdict needs `about.evaluation_id`; feedback by item uses an explicit `expected_outcome`.

**Find:** **observed** — Find “Imports labeled starting examples (seeds)” and “Say what the right answer was” (`import_seeds`, `submit_feedback`). Read each `inputSchema` before first use.

**Seed fields:** **from docs** — The structure inline (`questions`, `attributes`, `aggregations`, `result`) or a registered hash in `structure_hash`, never both, and `examples:[{item_id, state, attribute_values?, summary?, metadata?, claims, reason?, unsure?}]`. Each example needs an `expected_outcome` or `expected_answer`; a `verdict` belongs on feedback, not seeds. Seed submissions are atomic, allow up to 200 examples, and have a 60-second budget. **observed** (previous shape) — Ten examples cost about USD 0.00017 on Jev.

**Seed result:** **from docs** — `version: {structure_hash}`, `total`, `agreed`, and `results[]`, each with `item_id`, `evaluation_id`, `feedback_id`, `status`, `outcome`, `value`, `agreed`, and `conflicts` (`{question, expected, got}`). Save the returned evaluation ids.

**Seed cost:** **from docs** — Each result carries the same `cost` shape as evaluate; the top-level `cost` sums all four fields across the batch. Save the total with the batch result.

**Feedback fields:** **observed** — `about` is `{evaluation_id}` or `{decision_id,item_id}`; supply `claims`, `source`, and optional `reason`. **from docs** — `source:"owner"` is the decision owner's judgment and decides the case; `app_native` and `app_resolution` are weak feedback. Owner feedback requires `decisions:develop` according to the tool description. **observed** — The test key successfully submitted owner feedback.

**Gotchas:** **from docs** — `expected_answer` names a question only (an aggregation or attribute is a 400 listing the questions); `expected_outcome` is checked against the result's outcomes (400 listing them); a result that is a value (`status: "computed"`) takes no `expected_outcome` or `verdict`; one end-level claim per submission, never `verdict` with `expected_outcome`. Previews, suggestions and `mb seed` use the same System One structures and claims. **observed** — Feedback by item on a primary evaluation resolves it with `resolved_from_item:true`. A seed-only item returns 404 and directs you to `about.evaluation_id`. `agreed` is false when any answer conflicts even if the outcome matches; details are in `conflicts`.

**Done when:** Each submitted example or judgment has a recorded result, seed evaluation ids are retained, and any conflicts have been inspected.

## Backtest a change

**Goal:** Compare a candidate structure against seeded and reviewed cases before shipping it.

**Find:** **from docs** — Find “Replays a structure change over past cases with feedback” (`preview`). Read its `inputSchema` first.

**Request fields:** **from docs** — `structure` (the candidate, inline) is required; `suggestion_id` or `base: {structure_hash}` is optional. There are no `weights` or `outcomes`: the structure's result carries its outcomes, and it must be `bands`, `rules` or `match` (a value result is 400 `OUTCOMES_INVALID`). Without either, the base is the structure of the newest primary evaluation in the key's environment, else the newest seeded one.

**Result:** **from docs** — The response includes `base` and `candidate` (`{structure_hash, engine_profile}`), `evidence: {label, by_split:{seed,train,holdout}, action_mix, changed_items[], noise_rate}`, `cost`, `environment`, and `hashes_match` (`{structure}`, null without a suggestion). Changed items carry each side's `outcome`, `value` and per-question `answers`. Evidence labels, weakest first, are Starting examples, Training cases, and Held out; the docs also name Validated (cases reviewed after the change was frozen), but the API's label list does not include it today. Unchanged questions reuse stored answers only when engine, requested model, resolved model and adapter also match; other questions may spend engine calls.

**Gotchas:** **observed** — A held-out case listed in the response subsequently read back as `split:"burned"`. `changed_items` lists only fixes and breaks. Previewing leaves the candidate unregistered; register it or evaluate it inline before evaluating by hash. **from docs** — Changes to these behaviors are planned; re-check [Changing soon](verified.md#d-changing-soon-as-of-2026-10-27).

**Done when:** You have inspected the evidence label, split counts, fixes, breaks, cost, and environment, and can decide whether to ship the candidate. Registration only makes the hash available; ship by putting the new structure (or its hash) in the caller's code and evaluating with it.

## Ask for suggestions

**Goal:** Obtain a proposed structure change, inspect its evidence, and compare it with your own candidate.

**Find:** **from docs** — Find “Proposes one change to the decision's structure from its feedback” (`generate_suggestion`), plus its list/get/status siblings. Find “Apply a Moonbounce suggestion to this repo” for instructions and a diff the agent applies. Read each tool's `inputSchema` before first use.

**Request fields:** **from docs** — All optional: `kind` (`question_rewrite` or `new_question`) and a local `structure`, which must already be registered. The base's result must decide outcomes (400 `OUTCOMES_INVALID` otherwise).

**Result:** **from docs** — `{suggestion, reason?}`; `suggestion` is null with a reason such as “No change worth proposing yet: …”. A suggestion carries `kind`, `base` and `candidate` structure hashes, `candidate_structure`, a `diff` of JSON pointers under `/questions/<name>` and `/aggregations/<name>/inputs/-` with `before` values, and its frozen `evidence`. A `question_rewrite` changes one question's text and criteria (an object instruction keeps its `context`); a `new_question` adds a question and an input of the aggregation that feeds the result. **observed** — Generation on 11 seeds and two reviews returned `suggestion:null` with “no candidate fixed more than it broke on the cases it learned from.”

**Gotchas:** **observed** — Suggestions are separate per environment. **from code, reproduced locally** — Only feedback that decides an outcome makes a case to learn from: `source:"owner"` with `expected_outcome` (a `verdict` only on an evaluation that decided an outcome), a reviewer's answer, or a seed with `expected_outcome`. `app_native`/`app_resolution` feedback and `expected_answer` claims on their own never do. When no case is left for the writer, the null result's reason counts the held-out cases and what it left out and why. **from docs** — A test key's suggestions are never accepted; acceptance happens only with production keys, when an evaluation with the candidate structure hash first serves primary traffic. Production acceptance was not checked live. Treat a proposal as a lead and compare it against your own candidate through a backtest.

**Done when:** You have either recorded the null result and reason, or inspected a proposed change and its backtest evidence. When applying it, use the returned instructions and diff to edit the decision file (`{decision_id, structure}`) in the caller's repo.

## Pick an engine

**Goal:** Choose an offered engine and confirm the decision's binding and execution agree.

**Find:** **observed** — Find “The engines this deployment offers,” “The decision's current binding revision,” and “Sets the engine and model a decision runs on” (`list_engines`, `get_binding`, `set_binding`). Read each `inputSchema` before first use.

**Offered engines at the check:**

| Engine | Model | Pinning | Input price per million tokens |
| --- | --- | --- | --- |
| **observed** — `jev` | `jev-1.14.0` (default model) | Pinned | USD 0.042 |
| **observed** — `clef` | `@cf/cloudflare/clef` | Not pinned | USD 0.24 |

**Capabilities:** **from docs** — Both answer `noul` questions over a text or JSON state. **observed** — Image is marked “preview”; image behavior was not verified. Re-list engines to obtain current models, capabilities, and prices.

**Request fields:** **observed** — A binding write with body `{engine:"clef"}` returned 201, `created:true`, and revision 2; `model` is optional. First registration had created the default Jev binding at revision 1. After the switch, evaluation by structure hash used Clef, returned `version.binding:2`, and listed the engine per feature in `engines` (per question in the System One shape).

**Gotchas:** **observed** — Each environment has its own binding; try a switch with a test key first. **from docs** — A binding switch moves all of a decision's traffic at the next call, and the structure's weights and cuts were set against the old engine's answers; backtest on a test binding first.

**Done when:** The binding identifies the intended engine/model and an evaluation confirms its binding revision and per-question engine.

## Check cost

**Goal:** Read evaluation and preview spending for a decision and window in the intended environment.

**Find:** **observed** — Find “Engine cost of one decision's evaluations and previews over a window” (`get_usage`). Read its `inputSchema` first and use its decision/window selectors.

**Response fields:** **observed** — `environment`, `window`, `evaluations[]` grouped by `engine_profile` and `assignment_mode` (`primary`/`seed`), token counts, and `usd_micros: {total, mean, p50, p95}`; also `engines[]` and `previews: {runs, features_run, input_tokens, usd_micros}`. Amounts are in micro-dollars. **from docs** — Each group also carries `estimated_usd_micros` (the estimated part), and the report adds `other_spend[]` (attempts that never committed, suggestion generation) and a `total`.

**Per-request cost:** **from docs** — Evaluate (including dry runs), `get_evaluation`, and seeds report `cost` with the same fields as a preview. `engine_calls` counts upstream engine calls on evaluate and seeds; one call can answer several features (Jev compiles them). On previews it counts features sent to engines instead. Only previews use the feature cache today; evaluate and seeds report `cache_hits: 0`. `get_evaluation` omits `cost` if no usage row exists. `usd_micros` is millionths of a US dollar. Per-request `cost` is reported cost only; suggestion generation's own cost appears only in the usage report's `other_spend`.

**Scale and limits:** **observed** — The whole 2026-10-27 check, recorded as 13 evaluations, 11 seeds, and one preview, cost under USD 0.001. **from docs** — The report breaks out estimates for abandoned calls and lists evaluations that never committed under `other_spend`; `other_spend` also holds suggestion generation, and `total` adds everything recorded, according to the tool description.

**Done when:** You have recorded the environment, window, evaluation and preview costs, the other spend, and how much of the total is estimated.

## Retries for every write

**from docs** — Every write takes an `idempotency-key` argument. The same key and request replay the first result for 24 hours; the same key with a different request returns 422 `IDEMPOTENCY_KEY_REUSED`. Keys are scoped to organization plus environment across every operation. Send a key on each write and reuse it only for retries of that request. Done when a successful result is recorded under that request's key.

**from docs** — Fresh evaluate and seed responses always include `cost`. Dry runs require cost because they cannot replay. Idempotent replays of pre-cost snapshots may omit it until deployment plus 24 hours; a seed replay omits cost from both the summary and every result. A replay's cost, when present, is the original spend, not a second charge.

## Read back or list

**Goal:** Find existing decisions and inspect what was registered or evaluated.

**Find:** **from docs** — `list_decisions`, `list_structures`, `list_evaluations`, `get_structure` and `get_evaluation` have descriptions saying “Reads only.” Read the discovered schemas first.

**Request:** `list_decisions`, `list_structures` and `get_structure` accept either `decisions:evaluate` or `decisions:develop`; evaluation reads require `decisions:evaluate`. Lists accept `limit` (1–100, default 50) and `cursor`. Follow `next_cursor` while `has_more` is true. Evaluation filters include `item_id`, `assignment_mode` and inclusive `since` / exclusive `until`.

**Gotchas:** Decisions appear through a binding in the key's environment. Structure counts and lists use the org's shared registry for those decisions. Evaluations are environment-specific; lists omit content. `list_structures` items carry each structure's question hashes in `question_hashes`; `list_evaluations` items carry `status`, `value`, `outcome` and `version.structure_hash`. `register_structure` returns hashes without the structure; `get_structure` returns it as `structure`, in authored key order, with its hashes. `get_evaluation` returns the evaluate shape plus `assignment_mode`, the recorded `cost` when a usage row exists, and `content: {state, attribute_values?}` when stored, otherwise `content: null`. A stored JSON null is `{state: null}` inside `content`. The internal `store_state` setting defaults to true. `list_feedback` remains HTTP only.

**Done when:** The requested page or record is read in the intended environment; fetch a definition or state only when the task needs it.

## Dry run

**Goal:** Run an item without creating an evaluation or feedback target.

**Find:** **from docs** — `evaluate`, with `moonbounce.dry_run: true`.

**Request:** Use either an inline structure or an already registered hash in `moonbounce.structure_hash`. Omit `idempotency-key`: it is refused with 400 for dry runs.

**Gotchas:** This still spends engine calls and uses ordinary admission limits. It stores only its usage, a `dry_run` billable unit and, if the org has none yet, the org's row: no registration, binding, content key or answer cache is created. Usage totals appear under `unknown` profile/mode without a per-engine breakdown. Repeating the call can spend again. Reading or submitting feedback about a `dry_` id is refused; MCP may reject its pattern before calling the API. Use a normal evaluation when the result should be kept.

**Done when:** The response has `dry_run: true`, a `dry_` id, the intended `environment`, `cost` and the result; keep it locally if needed.
