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

**Goal:** Import labeled examples, or record judgments about existing evaluations, then check how well the version agrees with the labels.

**Shape:** **from docs** — Imported examples and feedback take the System One shape. Claims work at two levels: `expected_answer` `{question, value: true|false}` for one question, and `expected_outcome` `{value}` (one of the outcome names the result can produce) or `verdict` `{value: "approve"|"reject"}` for the decision; plus `input_broken` `{problem}`. The previous claim kinds (`outcome`, `target`, `feature`, `action`) are refused; update them before sending. A verdict needs `about.evaluation_id`; feedback by item uses an explicit `expected_outcome`.

**Find:** **from docs** — Find “Imports labeled examples” and “Say what the right answer was” (`import_examples`, `submit_feedback`). Read each `inputSchema` before first use.

**Import fields:** **from docs** — `POST /v1/decisions/{decision_id}/examples` (the decision comes from the path). The structure inline (`questions`, `attributes`, `aggregations`, `result`; registered on first sight, as evaluate does) or a registered hash in `structure_hash`, never both, and `examples:[{item_id, state, attribute_values?, summary?, metadata?, claims, label_source?, reason?, unsure?}]`, up to 200 per call, `item_id` unique in the call. Each example needs an `expected_outcome` or `expected_answer`; a `verdict` belongs on feedback, not on an example. `label_source` says who labeled it: `human`, `synthetic` (an agent wrote the label) or `unknown` (the default). Only `human` labels count toward the no-new-severe-mistake rule, and among near-identical items the case kept is the best-ranked label: owner > reviewer > `human` > `unknown` > `synthetic`. A dataset JSONL line is exactly one example, so `mb import <file>` uploads a file as it is. Before any engine call every example is checked: content the structure refuses is 400 `CONTENT_INVALID` (`param` `examples.<i>.state.<key>`), bad claims 400 at `examples.<i>.claims.<j>`. **observed** (under the earlier `import_seeds`, 2026-10-27) — Ten examples cost about USD 0.00017 on Jev.

**How it runs:** **from docs** — An import is `evaluate` over the examples as a stored run (each evaluation stored as an example, `assignment_mode` `example`) plus `submit_feedback` with each example's labels (source `import`), in one transaction, so an example is never stored without its labels. Failures are per example, as in an evaluate run: an example whose engines fail comes back `status: "failed"` with `error: {code, message, retryable}` and stores nothing, while the rest are stored. The call has a 60-second budget; examples it could not start fail with `NOT_STARTED`. Send the failed ones again in a new call with a new Idempotency-Key.

**Import result:** **from docs** — The run's response (201): `run_id` (`evr_`), `expires_at`, `dry_run: false`, `version` (`{structure_hash, engine_profile, binding}`), `counts`, `cost`, and `results[]`, one per example in order, each with `item_id`, `id` (the `evl_` evaluation id), `status`, `outcome` or `value`, `cost`, `feedback_id`, or `error` when it failed. There is no agreement count in the response: pass `run_id` to `loss_report({candidate: {run_ids: [run_id]}})` to see how the version agrees with the labels (loss per case, the costliest mistakes and suspect questions). Save the evaluation ids and the run id (runs expire after a day; the evaluations stay).

**Import cost:** **from docs** — Each result carries the same `cost` shape as evaluate (on a failed example, what its engines spent before failing); the top-level `cost` sums the call. Save it with the result.

**Feedback fields:** **observed** — `about` is `{evaluation_id}` or `{decision_id,item_id}`; supply `claims`, `source`, and optional `reason`. **from docs** — The reason is context for whoever writes the next change: name the question that got it wrong, quote the words in the content that show it, and state the rule that applies. A sentence is enough. Write the same kind of reason on imported examples. **from docs** — `source:"owner"` is the decision owner's judgment and decides the case; `app_native` and `app_resolution` are weak feedback. Owner feedback requires `decisions:develop` according to the tool description. **observed** — The test key successfully submitted owner feedback.

**Many labels at once:** **from docs** — `submit_feedback` takes one submission, or `{submissions: [...]}` with 1 to 200, each exactly as one call would send it. All are checked first, then stored in one transaction, or none are (a refusal's `param` starts `submissions.<i>`); the response is `{environment, results: [...]}` in order. A submission may supersede one sent earlier in the same call. Owner submissions still need `decisions:develop`.

**Gotchas:** **from docs** — `expected_answer` names a question only (an aggregation or attribute is a 400 listing the questions); `expected_outcome` is checked against the result's outcomes (400 listing them); a result that is a value (`status: "computed"`) takes no `expected_outcome` or `verdict`; one end-level claim per submission, never `verdict` with `expected_outcome`. Previews and `mb import` use the same System One structures and claims. Feedback by `item_id` binds to primary traffic only: an item that has only imported examples returns 404 naming the example's evaluation id, so address it by `about.evaluation_id`. **observed** — Feedback by item on a primary evaluation resolves it with `resolved_from_item:true`.

**Done when:** Each example or judgment has a recorded result, failed examples have been resent in a new call, the evaluation and run ids are retained, and you have read the `loss_report` over the import's runs.

## Backtest a change

**Goal:** Compare a candidate structure against imported and reviewed cases before shipping it.

**Find:** **from docs** — Find “Replays a structure change over past cases with feedback” (`preview`). Read its `inputSchema` first.

**Request fields:** **from docs** — `structure` (the candidate, inline) is required; `base: {structure_hash}` is optional. There are no `weights` or `outcomes`: the structure's result carries its outcomes, and it must be `bands`, `rules` or `match` (a value result is 400 `OUTCOMES_INVALID`). Without `base`, the base is the structure of the newest primary evaluation in the key's environment, else the newest one with imported examples.

**Result:** **from docs** — The response includes `base` and `candidate` (`{structure_hash, engine_profile}`), `evidence: {label, by_split:{example,train}, action_mix, changed_items[], noise_rate}`, `cost`, `environment` and `loss_report`: base and candidate priced by the decision's objective (balanced losses without one), with loss per case, macro-F1, cost by mistake, suspect question, missing criterion and cut, the costliest mistakes with their answers and fixing flips, and `accepted` with `reasons`. Keep a change by the same rule: `accepted` true, and only compare reports with the same `objective.effective_hash` ([refine.md](refine.md#keep-or-drop-a-change)). Changed items carry each side's `outcome`, `value` and per-question `answers`. Evidence labels, weakest first, are Imported examples (`imported_examples`, in the `example` split) and Training cases (`training_cases`); both are in-sample, since every reviewed case is training data the writer and the agent may see. The docs also name Validated (blind spot checks after the version ships), but the API's label list does not include it today. Unchanged questions reuse stored answers only when engine, requested model, resolved model and adapter also match; other questions may spend engine calls.

**Gotchas:** **observed** — `changed_items` lists only fixes and breaks. Previewing leaves the candidate unregistered; register it or evaluate it inline before evaluating by hash. **from docs** — Changes to these behaviors are planned; re-check [Changing soon](verified.md#d-changing-soon-as-of-2026-10-27).

**Done when:** You have inspected the evidence label, split counts, fixes, breaks, cost, and environment, and can decide whether to ship the candidate. Registration only makes the hash available; ship by putting the new structure (or its hash) in the caller's code and evaluating with it.

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

## Score a dataset

**Goal:** Judge a version over a labeled dataset under the decision's objective, compare it with another version, and find the items behind the loss.

**Find:** **from docs** — `evaluate` (its description covers a run: “**A run** (`items`)”) and “Judges one version's labeled results” (`loss_report`). Read each `inputSchema` before first use.

**Request fields:** **from docs** — A run is `evaluate` with `items` in place of `state`: the structure's fields at the top level as for one item (inline: a stored run registers it, a dry run doesn't) or `moonbounce.structure_hash`, `moonbounce.decision_id`, and `items` of `{item_id, state, attribute_values?, expected?, label_source?}`. Each item is evaluated exactly as it would be alone: send `moonbounce.dry_run: true` to score versions (it stores no evaluation), and leave it off only to store the items as real traffic. Each result carries its `id` (`dry_` or `evl_`), and `include: ["evaluation"]` adds everything one evaluate returns for it. The label is carried with the run, never used to evaluate and never recorded as feedback. `label_source` is `human` for a label a person gave or settled and `synthetic` for one an agent wrote; only human labels count toward the no-new-severe-mistake rule. Over MCP, both tools take the request under `body`. Send up to 200 items per call and keep each `run_id` (it answers 201). Items the call couldn't start in its time fail with `NOT_STARTED` (a case it ran out of time for, `ENGINE_UNAVAILABLE`; both `retryable`): send them again in a new run, with a new Idempotency-Key, and list that run beside the original in `loss_report`: a resent item's result replaces its failed one. Then `loss_report({candidate: {run_ids}, base: {run_ids}})` judges every run together and returns `loss_per_case`, `accepted` and `reasons`. Add `cases: {filter: {kind: "wrong"}}` (or `changed`, `severe`, `missing_criterion`, `{kind: "cell", expected, got?}`, `{kind: "question", question}`, `{kind: "near_cut", at}`) to page through items by loss; each roll-up row lists example `item_id`s.

**Stored cases:** **from docs** — To score a version on the decision's labeled cases (flow 2), send `cases` in place of `items`: `{evaluation_ids}` from `list_cases`, or its filters (`outcomes`, `structure_hash`, `label_source`, `since`, `until`) with `limit` and `cursor` (`cases.next_cursor` back for the next page). Always dry runs. Each case reuses the answers it stored for questions the version didn't change, so only new or reworded questions spend; `refresh` names questions to ask afresh. The run keeps each case's label, label source and answer corrections. A preview is exactly two such runs and a `loss_report`: the base with `refresh` set to the base questions the candidate changed or dropped, and the candidate, both with `cases.outcomes` set to the outcomes either version can produce and paged over the same cases (a preview judges the newest 500). Pages aren't a snapshot: a label given while you page can move a case between pages. It can differ on a case only one version can read.

**Gotchas:** **from docs** — Loss is how wrong a version is, weighted by the objective's severities and by outcome so each expected outcome counts equally; `cost` in a run's response is money (engine spend). Never pass the results back yourself: pass `run_ids`, so they don't go through your context. An item whose engines fail is that item's result (`status: "failed"`, `error.retryable`): `loss_report` leaves retryable failures out of both sides (`excluded.failed_transient`) and prices the rest as mistakes, so resend the retryable ones. Runs expire after a day. Price base and candidate on the same items: items the base lacks are counted in `excluded.no_base`. Each item that succeeded is billed as a dry run; a failed one's engine spend is still recorded.

**Done when:** `loss_report` shows the candidate's loss per case against the base's, with `accepted`, and you have looked at the worst cases it lists.

## List labeled cases

**Goal:** See the labels a decision has: which cases it gets wrong now, who labeled them, and what reviewers said.

**Find:** **from docs** — “Reads only. The decision's labeled cases in the key's environment” (`list_cases`). Read its `inputSchema` first.

**Request fields:** **from docs** — Path `decision_id`; query `limit` (default 50, at most 200, or 50 with `include=content`), `cursor`, and filters: `outcomes` (comma-separated without spaces, or a JSON array of strings when a name has a comma), `structure_hash` (the version a case ran under), `since` and `until` on when the evaluation ran, and `label_source` (`human`, `synthetic`, `unknown`). `structure_hash`, `since` and `until` choose which evaluations count before one case per item group is kept, so a case labeled under an older version still lists under that version's hash, and `excluded` counts only those evaluations. Needs `decisions:develop`.

**Result:** **from docs** — `cases`, newest label first, each with `evaluation_id`, `item_id`, `expected` (the label), `label_source`, `consensus` (`owner`, `majority`, `example`), `feedback_source`, `split`, `ran` (`structure_hash`, `engine_profile`, and the `status` and `outcome` it gave then), `answer_corrections` (`{question_hash, questions, expected}`), `reasons` (what people wrote), `feedback_ids`, `created_at`, `labeled_at`, and `content` only with `include=content`. Also `total` (matching cases), `excluded` (evaluations with feedback that are not cases, by reason), `next_cursor` and `has_more`.

**Gotchas:** **from docs** — These are the cases a preview replays: one per item group, with the same exclusions; set `outcomes` to the outcomes your versions produce, and a preview replays the newest 500 of that list. To score a version on them, send their `evaluation_id`s (or the same filters) to `evaluate` with `cases` ([Score a dataset](#score-a-dataset)). An imported example's `label_source` is the one it was imported with (`unknown` when none was sent). A case whose `ran.outcome` differs from `expected` is one the version it ran under got wrong. A cursor works only with the filters it was issued for. Content stays on the server unless asked for; reasons are often the best clue for a rewrite.

**Done when:** You have the cases you need, with the counts in `excluded` explaining what is missing.

## Set the objective

**Goal:** Store what the decision optimizes for, so previews and optimizers score against the same objective as the decision file.

**Find:** **from docs** — Find “Stores the decision file's `objective` as a new revision” and “The decision's current objective revision” (`set_objective`, `get_objective`). Read each `inputSchema` before first use.

**Request fields:** **from docs** — The objective lives in the decision file beside the structure: `{decision_id, structure, objective}`. Ask the person which mistakes matter most, in plain words, and write them as `mistakes` (`{expected, got, severity}`, with severity `minor`, `normal`, `costly` or `severe`, or a direct `loss`); add `fallback` for the safe outcome and an optional `target`. There are no limits on an outcome's share, recall or precision (an objective with `limits` is refused): scores weigh each expected outcome equally however many cases it has, so an example set's mix doesn't change them. A two-outcome decision can use `{"outcome": "<one>", "priority": "catch_all" | "be_sure" | "balanced"}`. Send `{"objective": <the file's objective>}`; `{"objective": null}` clears it.

**Gotchas:** **from docs** — The objective never changes `structure_hash`. Each environment keeps its own; push it with the key you preview with. 404 until the decision exists. Read `unknown_outcomes` in the response: an outcome the newest structure can't produce is left out when scoring, usually a typo or a renamed outcome. Without an objective the decision is scored with balanced losses. Previews and `loss_report` judge base and candidate by it.

**Done when:** `get_objective` (or loop status) shows the hash of the decision file's objective, and `unknown_outcomes` is empty. The hash is SHA-256 over the objective's canonical JSON, so compare with `mb objective`, which hashes the file locally, rather than computing it by hand.

## Check cost

**Goal:** Read evaluation and preview spending for a decision and window in the intended environment.

**Find:** **observed** — Find “Engine cost of one decision's evaluations and previews over a window” (`get_usage`). Read its `inputSchema` first and use its decision/window selectors.

**Response fields:** **observed** — `environment`, `window`, `evaluations[]` grouped by `engine_profile` and `assignment_mode` (`primary`/`seed` at the check; **from docs** now `primary`/`example`), token counts, and `usd_micros: {total, mean, p50, p95}`; also `engines[]` and `previews: {runs, features_run, input_tokens, usd_micros}`. Amounts are in micro-dollars. **from docs** — Each group also carries `estimated_usd_micros` (the estimated part), and the report adds `other_spend[]` (attempts that never committed, suggestion generation) and a `total`.

**Per-request cost:** **from docs** — Evaluate (including dry runs), `get_evaluation`, and `import_examples` report `cost` with the same fields as a preview. `engine_calls` counts upstream engine calls on evaluate and imports; one call can answer several features (Jev compiles them). On previews it counts features sent to engines instead. Only previews and runs over `cases` use the feature cache: evaluate, runs over `items` and imports report `cache_hits: 0`. `get_evaluation` omits `cost` if no usage row exists. `usd_micros` is millionths of a US dollar. Per-request `cost` is reported cost only.

**Scale and limits:** **observed** — The whole 2026-10-27 check, recorded as 13 evaluations, 11 imported examples (then seeds), and one preview, cost under USD 0.001. **from docs** — The report breaks out estimates for abandoned calls and lists evaluations that never committed under `other_spend`; `other_spend` also holds suggestion generation, and `total` adds everything recorded, according to the tool description.

**Done when:** You have recorded the environment, window, evaluation and preview costs, the other spend, and how much of the total is estimated.

## Retries for every write

**from docs** — Every write except `fill_review_queue` takes an `idempotency-key` argument. The same key and request replay the first result for 24 hours; the same key with a different request returns 422 `IDEMPOTENCY_KEY_REUSED`. Keys are scoped to organization plus environment across every operation. Send a key on each write and reuse it only for retries of that request. Done when a successful result is recorded under that request's key.

**from docs** — Fresh evaluate and `import_examples` responses always include `cost`. Dry runs require cost because they cannot replay. Idempotent replays of pre-cost snapshots may omit it until deployment plus 24 hours. A replay's cost, when present, is the original spend, not a second charge.

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
