# Tasks

Evidence convention: **observed** means run live against the production host with a test-environment key on 2026-10-27; **from docs** means read only (Moonbounce's guides or tool descriptions). Request fields and response shapes below use the current naming; use the discovered schema for each request.

Before each task, list tools, find the one whose description matches the goal, and read its `inputSchema` before first use. For REST, match the operation in the current `/openapi.json` and read its schema.

## Define a spec

**Goal:** Create the decision JSON in the caller's repo and obtain its structure hash.

**Find:** **observed** — No tool writes the spec. Find “Hash a decision spec ... without registering it” (`hash_spec`); this public check needs no key. Read its `inputSchema` first.

**Request shape:** **observed** — Supply the spec in the schema's expected field. The spec has `format: "moonbounce.spec/0"` and a `nodes` map containing:

- **observed** — Features: `{type:"boolean", prompt}`. Live features are boolean only; `prompt` asks the engine to detect the feature.
- **observed** — Combinations: `{type:"combine", terms:[{node, effect?: raises|lowers}], prior?: mean|any, fixed?:{link,weights,bias}}`.
- **observed** — Calibration: `{type:"calibrate", node}`.
- **observed** — `output` names the score node. `target` states the yes/no proposition whose probability the score represents; it is required for a combine/calibrate output and for feedback `target` claims. `guidance` and `meta` are optional.

**Gotchas:** **observed** — Hashing ignores key order, whitespace, NFC normalization, and `meta`; editing `prompt`, `guidance`, or `target` changes the structure hash. Mean-prior combinations dilute existing terms when adding a feature; `prior:"any"` can saturate near 1.0. Fixed weights must have signs matching their terms' effects or the server returns 400. **observed** — An old key (`question`, a term's `feature`, calibrate's `input`) gets a 400 naming the new one; the hashes are the same as before.

**Done when:** The JSON is in the caller's repo and the checking tool returns its structure hash.

## Register it

**Goal:** Make a structure available to evaluations by hash in the chosen environment.

**Find:** **observed** — Find “Stores a spec's structure under the decision so evaluations can name it by hash” (`register_structure`). Read its `inputSchema` first.

**Request:** **observed** — Provide the decision and spec using the current schema. Registration stores the spec's structure under that decision. A normal evaluation with an inline spec also registers it on first sight; a dry run does not.

**Gotchas:** **observed** — First registration in an environment creates the engine binding; the checked deployment selected Jev at revision 1. A backtest currently does not register its candidate. Register or evaluate inline before using a candidate's hash.

**Done when:** Registration succeeds and the structure hash is available for that decision in the intended environment.

## Evaluate

**Goal:** Score one item using the caller's spec and cut points.

**Find:** **observed** — Find “Scores one item with a spec (inline, or by registered hash)” (`evaluate`). Read its `inputSchema` first.

**Request fields:** **observed** — `decision_id`; either `spec` inline or `structure` hash; `weights` (`"default"` or a weights hash); `item_id`; `subject` parts such as `{type:"text",name,text}`; and `outcomes: {below, cuts:[{at,outcome}]}`.

**Result:** **observed** — The response has `id`, `status`, `outcome`, `score`, `nodes` with each node's `value`, `version: {structure, weights, weights_source, weights_fitted, binding, engine_profile}`, `engines`, and `usage`. Default weights returned `weights_fitted:false`. A mean combine of 0.99 and 0.29 scored 0.64.

**Gotchas:** **observed** — The budget is 10 seconds; the `prefer` argument can shorten it. **from docs** — The response carries `environment`; check it says the environment you meant. Preserve the evaluation `id` for feedback.

**Cost:** **from docs** — Fresh responses, including dry runs, carry `cost: {engine_calls, cache_hits, input_tokens, usd_micros}` with the same fields as a preview. `engine_calls` counts upstream engine calls on evaluate and seeds; one call can answer several features (Jev compiles them). On previews it counts features sent to engines instead. Calls, input tokens and money match the recorded usage. Only previews use the feature cache today; evaluate and seeds report `cache_hits: 0`. Keep `usage` for input and output token counts.

**Status:** **from docs** — `outcome` and `score` can be absent (an abstained or failed evaluation), and a node can report `abstained` or `failed` instead of a `value`. Check `status` before reading them.

**Done when:** A response identifies the evaluation, its `status`, the version used and, when the status is `scored`, the outcome and score.

## Add labeled data

**Goal:** Supply starting examples or judgments about existing evaluations, then inspect agreement and conflicts.

**Find:** **observed** — Find “Imports labeled starting examples (seeds)” and “Say what the right answer was” (`import_seeds`, `submit_feedback`). Read each `inputSchema` before first use.

**Seed fields:** **observed** — Supply `spec` or `structure`, `weights`, `outcomes`, and `examples:[{item_id, subject, claims:[{kind:"outcome"|"target"|"feature"|"input_broken", ...}]}]` (a `verdict` claim needs an evaluation whose outcome was shown, so it belongs on feedback, not seeds — **from docs**); take each claim's remaining fields from the schema. Seed submissions are atomic, allow up to 200 examples, and have a 60-second budget.

**Seed result:** **observed** — `total`, `agreed`, and `results[]`, each with `evaluation_id`, `feedback_id`, `outcome`, `score`, `agreed`, and `conflicts`. Ten examples cost about USD 0.00017 on Jev. Save the returned evaluation ids.

**Seed cost:** **from docs** — Each result carries the same `cost` shape as evaluate; the top-level `cost` sums all four fields across the batch. Save the total with the batch result.

**Feedback fields:** **observed** — `about` is `{evaluation_id}` or `{decision_id,item_id}`; supply `claims`, `source`, and optional `reason`. **from docs** — `source:"owner"` is the decision owner's judgment and decides the case; `app_native` and `app_resolution` are weak feedback. Owner feedback requires `decisions:develop` according to the tool description. **observed** — The test key successfully submitted owner feedback.

**Gotchas:** **observed** — Feedback by item on a primary evaluation resolves it with `resolved_from_item:true`. A seed-only item returns 404 and directs you to `about.evaluation_id`. `agreed` counts examples without either outcome or feature conflicts: a matching outcome with a contradictory feature claim returned `agreed:false`, feature details in `conflicts`, and `agreed:0`.

**Done when:** Each submitted example or judgment has a recorded result, seed evaluation ids are retained, and any conflicts have been inspected.

## Backtest a change

**Goal:** Compare a candidate against seeded and reviewed cases before shipping it.

**Find:** **observed** — Find “Replays a spec change over past cases with feedback” (`preview`). Read its `inputSchema` first.

**Request fields:** **observed** — `spec`, `weights`, and `outcomes` are all required; `suggestion_id` or `base` is optional.

**Result:** **observed** — The response includes `evidence: {label, by_split:{seed,holdout,...}, action_mix, changed_items[], noise_rate}`, `cost`, `environment`, and `hashes_match` (null without a suggestion). Changed items carry before/after scores. **from docs** — Evidence labels, weakest first, are Starting examples, Training cases, and Held out; the docs also name Validated (cases reviewed after the change was frozen), but the API's label list does not include it today.

**Gotchas:** **observed** — The replay covered seeded and reviewed cases. A held-out case listed in the response subsequently read back as `split:"burned"`. `changed_items` lists only fixes and breaks, leaving unchanged cases and the score distribution unavailable for cut-point tuning. Previewing leaves the candidate unregistered; register it or evaluate it inline before evaluating by hash. **from docs** — Changes to these behaviors are planned; re-check [Changing soon](verified.md#d-changing-soon-as-of-2026-10-27).

**Done when:** You have inspected the evidence label, split counts, fixes, breaks, cost, and environment, and can decide whether to ship the candidate. Registration only makes the hash available; ship by putting the new spec (or its hash) in the caller's code and evaluating with it.

## Ask for suggestions

**Goal:** Obtain a proposed spec change, inspect its evidence, and compare it with your own candidate.

**Find:** **observed** — Find “Proposes one change to the decision's spec from its feedback” (`generate_suggestion`), plus its list/get/status siblings. Find “Apply a Moonbounce suggestion to this repo” for instructions and a diff the agent applies. Read each tool's `inputSchema` before first use.

**Request fields:** **observed** — A `spec` sent to generation must already be registered. Supply `outcomes` unless the base recorded cut points; get the other selectors from the current schema.

**Result:** **observed** — Generation on 11 seeds and two reviews returned `{suggestion:null, reason}` with “No change worth proposing yet: no candidate fixed more than it broke on the cases it learned from.” **from docs** — The tool description allows “a feature rewrite or a new feature.”

**Gotchas:** **observed** — Suggestions are separate per environment. **from docs** — A test key's suggestions are never accepted; acceptance happens only with production keys. Production acceptance was not checked live. Treat a proposal as a lead and compare it against your own candidate through a backtest.

**Done when:** You have either recorded the null result and reason, or inspected a proposed change and its backtest evidence. When applying it, use the returned instructions and diff to edit the caller's repo.

## Pick an engine

**Goal:** Choose an offered engine and confirm the decision's binding and execution agree.

**Find:** **observed** — Find “The engines this deployment offers,” “The decision's current binding revision,” and “Sets the engine and model a decision runs on” (`list_engines`, `get_binding`, `set_binding`). Read each `inputSchema` before first use.

**Offered engines at the check:**

| Engine | Model | Pinning | Input price per million tokens |
| --- | --- | --- | --- |
| **observed** — `jev` | `jev-1.14.0` (default model) | Pinned | USD 0.042 |
| **observed** — `clef` | `@cf/cloudflare/clef` | Not pinned | USD 0.24 |

**Capabilities:** **observed** — Both answer boolean features over text and JSON. Image is marked “preview”; image behavior was not verified. Re-list engines to obtain current models, capabilities, and prices.

**Request fields:** **observed** — A binding write with body `{engine:"clef"}` returned 201, `created:true`, and revision 2; `model` is optional. First registration had created the default Jev binding at revision 1. After the switch, evaluation by structure hash used Clef, returned `version.binding:2`, and listed the engine per node in `engines`.

**Gotchas:** **observed** — Each environment has its own binding; try a switch with a test key first. **from docs** — A decision that pins weights fit for one engine profile keeps running on the newest binding revision that profile fits, and gets 409 `WEIGHTS_INCOMPATIBLE` only when no revision fits; default weights fit any engine and move at once.

**Done when:** The binding identifies the intended engine/model and an evaluation confirms its binding revision and per-node engine.

## Check cost

**Goal:** Read evaluation and preview spending for a decision and window in the intended environment.

**Find:** **observed** — Find “Engine cost of one decision's evaluations and previews over a window” (`get_usage`). Read its `inputSchema` first and use its decision/window selectors.

**Response fields:** **observed** — `environment`, `window`, `evaluations[]` grouped by `engine_profile` and `assignment_mode` (`primary`/`seed`), token counts, and `usd_micros: {total, mean, p50, p95}`; also `engines[]` and `previews: {runs, features_run, input_tokens, usd_micros}`. Amounts are in micro-dollars.

**Per-request cost:** **from docs** — Evaluate (including dry runs), `get_evaluation`, and seeds report `cost` with the same fields as a preview. `engine_calls` counts upstream engine calls on evaluate and seeds; one call can answer several features (Jev compiles them). On previews it counts features sent to engines instead. Only previews use the feature cache today; evaluate and seeds report `cache_hits: 0`. `get_evaluation` omits `cost` if no usage row exists. `usd_micros` is millionths of a US dollar. Suggestion generation's own cost is still not reported; do not treat usage totals as including it.

**Scale and limits:** **observed** — The whole 2026-10-27 check, recorded as 13 evaluations, 11 seeds, and one preview, cost under USD 0.001. **from docs** — Totals under-count: abandoned calls and uncommitted evaluations record no usage, according to the tool description.

**Done when:** You have recorded the environment, window, evaluation and preview costs, with the accounting limitation attached.

## Retries for every write

**from docs** — Every write takes an `idempotency-key` argument. The same key and request replay the first result for 24 hours; the same key with a different request returns 422 `IDEMPOTENCY_KEY_REUSED`. Keys are scoped to organization plus environment across every operation. Send a key on each write and reuse it only for retries of that request. Done when a successful result is recorded under that request's key.

**from docs** — Fresh evaluate and seed responses always include `cost`. Dry runs require cost because they cannot replay. Idempotent replays of pre-cost snapshots may omit it until deployment plus 24 hours; a seed replay omits cost from both the summary and every result. A replay's cost, when present, is the original spend, not a second charge.

## Read back or list

**Goal:** Find existing decisions and inspect what was registered or evaluated.

**Find:** **from docs** — `list_decisions`, `list_structures`, `list_evaluations`, `get_structure` and `get_evaluation` have descriptions saying “Reads only.” Read the discovered schemas first.

**Request:** `list_decisions`, `list_structures` and `get_structure` accept either `decisions:evaluate` or `decisions:develop`; evaluation reads require `decisions:evaluate`. Lists accept `limit` (1–100, default 50) and `cursor`. Follow `next_cursor` while `has_more` is true. Evaluation filters include `item_id`, `assignment_mode` and inclusive `since` / exclusive `until`.

**Gotchas:** Decisions appear through a binding in the key's environment. Structure counts and lists use the org's shared registry for those decisions. Evaluations are environment-specific; lists omit content. `register_structure` returns hashes and metadata without `spec`; `get_structure` returns `spec` in the current vocabulary and authored order (older rows may have lost that order). `get_evaluation` returns `assignment_mode`, the recorded `cost` when a usage row exists, and `subject` when stored; absent subject means content was not stored (`store_content: false`). `list_feedback` remains HTTP only.

**Done when:** The requested page or record is read in the intended environment; fetch a spec or subject only when the task needs it.

## Dry run

**Goal:** Score an item without creating an evaluation or feedback target.

**Find:** **from docs** — `evaluate`, with `body.dry_run: true`.

**Request:** Use either inline `spec` or an already registered `structure`. Omit `idempotency-key`: it is refused with 400 for dry runs.

**Gotchas:** This still spends engine calls and uses ordinary admission limits. Only usage is stored: no registration, binding, content key or feature cache is created. Usage totals appear under `unknown` profile/mode without a per-engine breakdown. Repeating the call can spend again. Reading or submitting feedback about a `dry_` id is refused; MCP may reject its pattern before calling the API. Use a normal evaluation when later feedback should target the result.

**Done when:** The response has `dry_run: true`, a `dry_` id, the intended `environment`, `cost` and the result; keep it locally if needed.
