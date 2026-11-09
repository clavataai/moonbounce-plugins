# Tasks

These sections say how the tools work and what to watch for. The tool descriptions and input schemas are the source of truth: read them first, and follow them where this file differs.

Before each task, list tools, find the one whose description matches the goal, and read its `inputSchema` before first use. For REST, match the operation in the current `/openapi.json` and read its schema.

## Define a structure

**Goal:** Create the decision file in the caller's repo and obtain its structure hash. How to design it (questions or attributes, which aggregation): [Design a structure](design.md).

**Find:** No tool writes the file. Upload it to check it ([Upload a file](#upload-a-file), `kind: "structure"`): the upload validates it and answers its hashes, registering nothing.

**Shape:** The file is the decision file `{decision_id, structure, objective?}`, and its `structure` is:

- `questions`: `{type:"noul", instructions, criteria?:{true?,false?}, reads?:["state.<key>", "state.<key>.<key>", …]}` (object paths at any depth, sent nested as in the state; array indexes such as `state.items[0]` are a 400, and so is an entry repeated or containing another, like `state.order` with `state.order.id`). Its answer is P(true). `{type:"choice", instructions, criteria:{<option>: <description or null>}, reads?}` and `{type:"score", instructions, criteria:[<level 0>, <level 1>, …], reads?}` are evaluated too (answers `{type:"choice", choice, probabilities, confidence}` or `{type:"score", score, probabilities, legend, confidence}`; a reference reads one's `option` or `confidence`; an `expected_answer` on one is an option name or a level number), and the loss report flips them to each option or level (`flip_to`). `instructions` is a string, object or array, sent as written. Without `reads` a question sees the whole state.
- `attributes` (optional): typed values aggregations compare exactly. The model never gets the attribute itself, but a derived one's source field stays in `state`, so a question that reads it (or has no `reads`) still sees it: `{type:"string", enum?, default?, source?, from?}` (without `enum`, open text of at most 200 characters, read only by `is` with another feature and by `abstained`; an empty value is unknown), `{type:"number", minimum?, maximum?, default?, source?, from?}` or `{type:"boolean", default?, source?, from?}`. With `from` (a state path, as in `reads`), evaluate fills it from the call's `state`, validated by the declaration with no coercion; a missing or null path takes `default`, else a 400 at the path. Without `from`, the caller sends it per call in `attribute_values`. A fact already in the state is a derived attribute; `attribute_values` is for facts the app computes.
- `aggregations` (optional): `weighted_mean` (positive weights or none, `invert`), `logistic` (`bias`, signed `weight` on every input), `min`/`max`/`product`/`noisy_or` (`invert`), checks `all`/`any` (each input a rule condition's test: `gte`/`gt`/`lte`/`lt` with default ≥ 0.5, `is`/`in`, `abstained`, unique by the whole test), and outcome types `bands` (`input`, `below`, ascending `cuts:[{at,outcome}]`, `abstain?`), `rules` (`rules:[{when,outcome}]`, `else`) and `match` (`input`, `cases`, `else?`, `abstain?`). Numeric and check types list `inputs:[{name,...}]`; `bands`/`match` read `input`, and `rules` read their conditions. Every reference (`inputs[].name`, `input`, a rule's `name`, `result`) is a path: `questions.<name>`, `attributes.<name>` or `aggregations.<name>`. Where a test takes a constant (`is`, `gte`, `gt`, `lte`, `lt`) it also takes another feature, `{name, option?, confidence?}` (a feature comparison, unknown when either side is), and `is` with one takes `normalize: "case" | "email"`, applied to both sides.
- `result`: the final feature's path (`questions.x`, `attributes.x` or `aggregations.x`).

**Gotchas:** Names are lowercase slugs, unique across questions, attributes and aggregations. A bare name, a path under the wrong root or an unknown name where a reference goes is a 400 naming the path; the map keys, `answers`, `aggregations` and claims stay bare. Hashing ignores key order, NFC normalization of instructions, criteria and attribute sources, the order of `reads` and `inputs`, and spelled-out defaults (`invert:false`, `gte:0.5`, empty `criteria`, all-1 `weighted_mean` weights). Read paths, string enum labels/defaults and label comparisons stay exact, since customer keys and values are not normalized; a feature operand and `normalize` are hashed as written, and a test's sides keep their order. Any other edit, including a weight, threshold or cut point, is a new hash. Each question also gets its own hash (its definition without its name); [Version hashes](https://docs.moonbounce.ai/learn/version-hashes) has the algorithm. Cycles between aggregations are a 400 that names the cycle. There is no weights hash.

**Done when:** The JSON is in the caller's repo and its upload answers its structure hash.

## Upload a file

**Goal:** Send a dataset file or a structure once, so later calls name it instead of carrying it.

**Find:** `create_upload` (`POST /v1/decisions/{decision_id}/uploads`). Read its `inputSchema` first.

**Steps:**

1. Call `create_upload` with `{kind: "items"}` for a dataset JSONL or `{kind: "structure"}` for a decision file. It answers a `url` that takes one file until `expires_at` (15 minutes). It takes no idempotency key: a retry makes another upload, and an unused one expires.
2. Send the file with one plain command, the `url` as it came (it has no `?`, so it needs no quoting): `curl -sS -X PUT --data-binary @moonbounce/<decision_id>.examples.jsonl <url>`.
3. Read the JSON curl prints. For items: `upload_id`, `lines` and `labeled` (the lines with an `expected_outcome` claim). For a structure: `structure_hash`, `question_hashes` and `warnings` (as `POST /v1/validate` gives them: a question, attribute or aggregation the result never reaches).

**Refusals:** Nothing is kept unless every line is valid: 400 `UPLOAD_INVALID` lists the first 20 problems in `details.issues`, each message starting `line <n> (<item_id>):` (lines counted from 1) and its `param` the line's field. A line over 1 MB as sent, or over 2 MB as stored (each `,` and `:` a byte more, and an exponent spelled out: `1e-300` is 302 bytes), is one of them (`LINE_TOO_LONG`). A structure that doesn't validate is 400 `CONTENT_INVALID` with its issues. Fix the file and send it to the same `url` again; once that has expired (404 `UPLOAD_UNKNOWN`), call `create_upload` again. 409 `UPLOAD_RECEIVED`: the file is already in, and `details.receipt` is what its first send answered. 409 `UPLOAD_BUSY`: a send of it is under way, or one died; if none of yours is running, call `create_upload` again. 413 `UPLOAD_TOO_LARGE`: over 50,000 lines or 50 MB (a structure, 1 MB). 409 `UPLOADS_LIMIT`: the team holds 200 items uploads (or 1,000 structure uploads) that haven't expired; name an upload you already have, send small content in the call itself, or wait for some to expire. A decision that keeps no content refuses `kind: "items"` (400 `CONTENT_NOT_KEPT`).

**Items:** Each line is one `import_examples` example, the dataset file's own format, with `claims` optional; blank lines are skipped, and an `item_id` appears once. A run or an import takes a slice as `items_from: {upload_id, offset, limit?}`: `limit` lines (200 by default, and at most) from line `offset` (counted from 0), fewer at the end of the file. A file of N lines is ceil(N/200) calls, at offsets 0, 200, 400…. A slice whose lines take more than 2 MB as stored is 422 `ITEMS_TOO_LARGE`: send a smaller `limit` and step the offsets by it.

**A structure:** The file is a decision file or a bare structure; only `structure` is read, and a `decision_id` other than the upload's is a 400. While the upload lives, dry runs (`evaluate` with `dry_run: true`, `create_run`) name it by its `structure_hash`, and `register_decision` registers it by that hash. A stored evaluation or run and an import need it registered.

**Lifetime:** An upload belongs to its decision and the key's environment, and lives about a day after it's received (`expires_at` in curl's answer), as a run does. Upload the file again after that, or once it changes.

**Done when:** curl's answer carries the `upload_id` or the `structure_hash`.

## Register it

**Goal:** Register the decision file's structure as a version of the decision, so evaluations can name it by hash.

**Find:** Find “Register a decision's version” (`register_decision`). Read its `inputSchema` first.

**Request:** Upload the decision file ([Upload a file](#upload-a-file)), then send `{decision_id, structure_hash}` with the hash it answered, so the structure is never written out in a call. An unknown `decision_id` creates the decision. 201 when this call added a version, 200 when that structure was already registered; the response is `{decision_id, version: {structure_hash, number, question_hashes, created_at}, created}`, not the structure object. `number` is the version's number under the decision (1, 2, 3 in registration order); the hash is its identity. The hash may also be one a live run of the decision ran. Any other hash is 404 `STRUCTURE_UNKNOWN`, and a copy that no longer validates is 400 `CONTENT_INVALID`. It takes no `objective`: store that with `set_objective`. A normal evaluation with an inline structure also registers it on first sight; a dry run does not.

**Gotchas:** First registration in an environment stores the deployment's default model as that environment's model setting, at revision 1; a later change of the default never moves the decision. A backtest does not register its candidate.

**Done when:** Registration succeeds and the version's hash and number are returned for that decision.

## Evaluate

**Goal:** Run one item through the caller's structure.

**Find:** Find “Evaluates one item against a decision” (`evaluate`). Read its `inputSchema` first; its description ends with a worked example `body`.

**Request fields:** `state` (any JSON; an object when questions have `reads`); the structure inline (`questions`, `attributes?`, `aggregations?`, `result`) or its hash in `moonbounce.structure_hash`, not both; `attribute_values` for declared attributes without `from` (a value for a derived one is a 400); optional `model` (an assertion: it must be the model the decision's setting names, or the call is a 400); and `moonbounce: {decision_id, item_id, structure_hash?, metadata?, summary?, dry_run?}`.

**Result:** `id`, `status`, `value`, `outcome`, `missing` (the result's questions with no answer, as `questions.x`; absent when none, so gate on `status === "decided" && !missing` before acting on full evidence), `answers` (by `type`: `{type:"noul", noul}`, `{type:"choice", choice, probabilities, confidence}` or `{type:"score", score, probabilities, legend, confidence}`, or `{type, abstained:true}` / `{type, failed:true}`; skip a type you don't know), `aggregations` (each `value`, or `outcome`/`via`, `missing` (input paths), `abstained`), `moonbounce` (a pure echo of what you sent), `version: {structure_hash, number}` (`number` is null when the structure isn't registered, as in a dry run of a new structure), `model: {id, resolved_model, revision, profile}` (`id` is the setting's model, copyable into `set_model`; `revision` is null when nothing is stored in the environment and the deployment default applied; `profile` is opaque: compare it for equality, never parse it) and `cost`, whose `calls` lists one entry per upstream model call (`questions`, the `reads` paths it sent, `input_tokens`). The result aggregation inside `aggregations` carries its own `via`.

**Gotchas:** Questions that read the same paths share one model call; a key nobody reads is never sent. An unknown hash is 404 `STRUCTURE_UNKNOWN` (a dry run also takes the hash of a structure a live upload or run of the decision holds); a missing read path (any level, or a non-object on the way) or a bad attribute value is 400 `CONTENT_INVALID` listing every issue. A question on the result's path that fails is 502 `ENGINE_FAILED` (503 when transient); one off the path reports `failed`. The budget is 10 seconds; the `prefer` argument can shorten it. The response carries `environment`; check it says the environment you meant. Preserve the evaluation `id`.

**Cost:** Fresh responses, including dry runs, carry `cost: {model_calls, questions, input_tokens, output_tokens, usd_micros, calls}`, the one cost shape every response uses. `model_calls` counts upstream model calls; `questions` counts the questions sent to a model; `cost.calls` shows which questions and keys each call carried. `cache_hits` appears only when the answer cache served a question.

**Dry run:** A dry run accepts an `idempotency-key` and ignores it, so a retried dry run runs, and is billed, again.

**Status:** `decided` (an outcome aggregation named `outcome`; `bands` adds the `value` it cut, `rules`/`match` add `via`), `computed` (the result is a number in `value`) or `abstained` (no answer; `outcome` is the `abstain` outcome when one is set). `via` is a zero-based rule index, `cases.<exact label>` for a matched case (so a case named `else` is `cases.else`), or `else` for the fallback. Check `status` before reading the rest.

**Done when:** A response identifies the evaluation, its `status`, the version used and the outcome or value.

## Add labeled data

**Goal:** Import labeled examples, or record judgments about existing evaluations, then check how well the version agrees with the labels.

**Shape:** Claims work at two levels: `expected_answer` `{question, value}` for one question (`value` is the answer's label: `true`/`false` for a noul, an option name for a choice, a level as a JSON number (`2`, not `"2"`) for a score; anything else is a 400 listing what it may be), and `expected_outcome` `{value}` (one of the outcome names the result can produce) or `verdict` `{value: "approve"|"reject"}` for the decision; plus `input_broken` `{problem}`. The previous claim kinds (`outcome`, `target`, `feature`, `action`) are refused; update them before sending. A verdict needs `about.evaluation_id`; feedback by item uses an explicit `expected_outcome`.

**Find:** Find “Imports labeled examples” and “Say what the right answer was” (`import_examples`, `submit_feedback`). Read each `inputSchema` before first use.

**Import fields:** `POST /v1/decisions/{decision_id}/examples` (the decision comes from the path). The structure inline (`questions`, `attributes`, `aggregations`, `result`; registered on first sight, as evaluate does) or a registered hash in `structure_hash`, never both, and `examples:[{item_id, state, attribute_values?, summary?, metadata?, claims, label_source?, reason?, unsure?}]`, up to 200 per call, `item_id` unique in the call. Each example needs an `expected_outcome` or `expected_answer`; a `verdict` belongs on feedback, not on an example. `label_source` says who labeled it: `human`, `synthetic` (an agent wrote the label) or `unknown` (the default). Only `human` labels count toward the no-new-severe-mistake rule, and among near-identical items the case kept is the best-ranked label: owner > reviewer > `human` > `unknown` > `synthetic`. A dataset JSONL line is exactly one example, so a dataset file goes by its upload ([Upload a file](#upload-a-file)): `items_from: {upload_id, offset}` in place of `examples`, one call per 200-line slice (offsets 0, 200, 400…), each line the example as written, `reason` and `metadata` included. A line without a label is a 400 naming its `item_id`. Before any model call every example is checked: content the structure refuses is 400 `CONTENT_INVALID`, and bad claims a 400, each naming the example.

**How it runs:** An import evaluates the examples as a stored run does (`create_run` without `dry_run`) (each evaluation stored as an example, `assignment_mode` `example`) plus `submit_feedback` with each example's labels (source `import`), in one transaction, so an example is never stored without its labels. Failures are per example, as in an evaluate run: an example whose model calls fail comes back `status: "failed"` with `error: {code, message, retryable}` and stores no evaluation and no feedback, while the rest are stored. The run still keeps every example's outcome label, failed ones too, until it expires (about a day). The call has a 60-second budget; examples it could not start fail with `NOT_STARTED`. Send the failed ones again in a new call with a new Idempotency-Key.

**Import result:** The run's response (201): `run_id` (`evr_`), `expires_at`, `dry_run: false`, `version` (`{structure_hash, number}`), `model` (`{id, resolved_model, revision, profile}`), `counts`, `cost`, and `results[]`, one per example in order, each with `item_id`, `id` (the `evl_` evaluation id), `status`, `outcome` or `value`, `cost`, `feedback_id`, or `error` when it failed. There is no agreement count in the response: pass `run_id` to `loss_report({candidate: {run_ids: [run_id]}})` to see how the version agrees with the labels (loss per case, the costliest mistakes and suspect questions). Save the evaluation ids and the run id (runs expire after a day; the evaluations stay).

**Import cost:** Each result carries the same `cost` shape as evaluate (on a failed example, what its model calls spent before failing); the top-level `cost` sums the call. Save it with the result.

**Feedback fields:** `about` is `{evaluation_id}` or `{decision_id,item_id}`; supply `claims`, `source`, and optional `reason`. The reason is context for whoever writes the next change: name the question that got it wrong, quote the words in the content that show it, and state the rule that applies. A sentence is enough. Write the same kind of reason on imported examples. `source:"owner"` is the decision owner's judgment and decides the case; `app_native` and `app_resolution` are weak feedback. Owner feedback requires `decisions:develop` according to the tool description. The test key successfully submitted owner feedback.

**Many labels at once:** `submit_feedback` takes one submission, or `{submissions: [...]}` with 1 to 200, each exactly as one call would send it. All are checked first, then stored in one transaction, or none are (a refusal's `param` starts `submissions.<i>`); the response is `{environment, results: [...]}` in order. A submission may supersede one sent earlier in the same call. Owner submissions still need `decisions:develop`.

**Gotchas:** `expected_answer` names a question only (an aggregation or attribute is a 400 listing the questions); `expected_outcome` is checked against the result's outcomes (400 listing them); a result that is a value (`status: "computed"`) takes no `expected_outcome` or `verdict`; one end-level claim per submission, never `verdict` with `expected_outcome`. Feedback by `item_id` binds to primary traffic only: an item that has only imported examples returns 404 naming the example's evaluation id, so address it by `about.evaluation_id`. Feedback by item on a primary evaluation resolves it with `resolved_from_item:true`.

**Done when:** Each example or judgment has a recorded result, failed examples have been resent in a new call, the evaluation and run ids are retained, and you have read the `loss_report` over the import's runs.

## Backtest a change

**Goal:** Compare a candidate structure against imported and reviewed cases before shipping it.

**Find:** Find “Replays a structure change over past cases with feedback” (`create_preview`). Read its `inputSchema` first.

**Request fields:** `structure` (the candidate, inline) is required; `base: {structure_hash}` is optional. There are no `weights` or `outcomes`: the structure's result carries its outcomes, and it must be `bands`, `rules` or `match` (a value result is 400 `OUTCOMES_INVALID`). Without `base`, the base is the structure of the newest primary evaluation in the key's environment, else the newest one with imported examples.

**Result:** The response includes its own `id` (`prv_`), `base` and `candidate` (`{version: {structure_hash, number}, model: {profile}}`, `number` null for a structure nobody registered), `evidence: {label, by_split:{example,train}, action_mix, changed_items[], noise_rate}`, `cost`, `environment` and `loss_report`: base and candidate priced by the decision's objective (balanced losses without one), with loss per case, macro-F1, cost by mistake, suspect question, missing criterion and cut, the costliest mistakes with their answers and fixing flips, and `accepted` with `reasons`. Keep a change by the same rule: `accepted` true, and only compare reports with the same `objective.effective_hash` ([refine.md](refine.md#keep-or-drop-a-change)). Changed items carry each side's `outcome`, `value` and per-question `answers`. Evidence labels, weakest first, are Imported examples (`imported_examples`, in the `example` split) and Training cases (`training_cases`); both are in-sample, since every reviewed case is training data the writer and the agent may see. Unchanged questions reuse stored answers only when the requested model, resolved model and adapter also match the model in effect in the key's environment (that environment's own setting, else the deployment default; a test key never uses production's); other questions may spend model calls.

**Gotchas:** `changed_items` lists only fixes and breaks. Previewing leaves the candidate unregistered; register it or evaluate it inline before evaluating by hash.

**Done when:** You have inspected the evidence label, split counts, fixes, breaks, cost, and environment, and can decide whether to ship the candidate. Registration only makes the hash available; ship by putting the new structure (or its hash) in the caller's code and evaluating with it.

## Pick a model

**Goal:** Choose an offered model and confirm the decision's setting and execution agree.

**Find:** Find “List the models a decision can run on,” “Get the model a decision runs on,” and “Choose the model a decision runs on” (`list_models`, `get_model`, `set_model`). Read each `inputSchema` before first use.

**Capabilities:** `list_models` gives `{default, models: [{id, resolved_model, pinned, question_types, modalities, pricing}]}`, the models this deployment offers; read it rather than assuming. Every model answers `noul`, `choice` and `score` questions over a text or JSON state (its `question_types`). An image capability marked “preview” is untested.

**Request fields:** `set_model({decision_id, model})` takes the `id` from `list_models` and always answers 200 with the setting plus `changed` (false when that model was already in effect). `get_model` returns `{decision_id, environment, model, pinned, revision, source}`: `source` is `setting` when a revision is stored in the key's environment, or `default` with `revision` null when nothing is stored and the deployment default applies. It never 404s for a decision that exists. The model in effect is that environment's own setting, else the deployment default. The decision's first registration in an environment stores the default as revision 1. An evaluation reports what ran in `model: {id, resolved_model, revision, profile}`.

**Choosing:** Keep the default unless the person has a reason to change it (cost, a model they need, image input). A switch changes every answer, so the structure's weights and cuts were tuned for the old model: score the last kept version again on the new one before relying on it.

**Gotchas:** Each environment has its own setting, and a test key never falls back to production's; try a switch with a test key first. A switch moves all of a decision's traffic at the next call. Backtest on a test setting first.

**Done when:** `get_model` identifies the intended model and an evaluation's `model.id` and `model.revision` confirm it.

## Score a dataset

**Goal:** Judge a version over a labeled dataset under the decision's objective, compare it with another version, and find the items behind the loss.

**Find:** “Run a version over many items” (`create_run`, `POST /v1/decisions/{decision_id}/runs`) and “Judge results under a decision's objective” (`loss_report`). Read each `inputSchema` before first use.

**Request fields:** `create_run` takes the decision in the path (`decision_id`) and a flat body, like `import_examples`, with no `moonbounce` wrapper: the structure's fields inline (`questions`, `attributes`, `aggregations`, `result`; a stored run registers it, a dry run doesn't) or `structure_hash` (on a dry run, also an upload's: [Upload a file](#upload-a-file)), then `items` of `{item_id, state, attribute_values?, expected?, label_source?}`, `items_from` or `cases` (below), with optional `dry_run`, `include`, `refresh` and `results`. Each item is evaluated exactly as it would be alone: send `dry_run: true` to score versions (it stores no evaluation), and leave it off only to store the items as real traffic. Each result carries its `id` (`dry_` or `evl_`), and `include: ["evaluation"]` adds everything one evaluate returns for it. A run you only score takes `results: "failed"`: the response lists only the failed items' results (`"none"`: none), while `counts`, `cost` and the run itself cover every item. The label is carried with the run, never used to evaluate and never recorded as feedback. `label_source` is `human` for a label a person gave or settled, `synthetic` for one an agent wrote, and `unknown` when nobody knows (the default); only human labels count toward the no-new-severe-mistake rule. Over MCP, `create_run` takes the request under `body`. Send up to 200 items per call and keep each `run_id` (it answers 201). Items the call couldn't start in its time fail with `NOT_STARTED` (a case it ran out of time for, `ENGINE_UNAVAILABLE`; both `retryable`; the reason code keeps that name): send them again in a new run, with a new Idempotency-Key, and list that run beside the original in `loss_report`: a resent item's result replaces its failed one. Then `loss_report({candidate: {run_ids}, base: {run_ids}})` judges every run together and returns `loss_per_case`, `accepted` and `reasons`. Over MCP it answers the summary (the verdict, each side's headline numbers and three costliest mistake cells, `changes` as counts, `excluded`, `report_url`) unless you send `detail: "full"`; with `show: true` beside `body`, a host with MCP Apps charts the whole report beside it (send it only for the report you show the person); over REST the default is `full`. Add `cases: {filter: {kind: "wrong"}}` (or `changed`, `severe`, `missing_criterion`, `{kind: "cell", expected, got?}`, `{kind: "question", question}`, `{kind: "near_cut", at}`, or `{kind: "like", item_id}` for the rows nearest one, nearest first) to page through items, or `cases: {filter, sample: n}` for n of them split across expected outcomes and spread over the candidate's answers, the same each time; each roll-up row lists example `item_id`s. How much to read, and how: [Looking at the data](refine.md#looking-at-the-data).


**A dataset file:** upload it once ([Upload a file](#upload-a-file)), then send `items_from: {upload_id, offset}` in place of `items`, one run per 200-line slice (offsets 0, 200, 400…), and list every run's id on its side of `loss_report`. Each line is an item: its `state`, `attribute_values` and `label_source`, and its `expected_outcome` claim as `expected` (a line without one isn't scored by outcome). A run of `items_from` is always a dry run and needs `decisions:develop`. New questions read the lines' `state`, and a new attribute works when it has a `default` or fills itself from the state (`from`); a version that needs content the lines lack (a caller-supplied attribute without a default, a state field) is 400 `CONTENT_INVALID` naming the item, so the file changes and goes up again. `loss_report` takes up to 50 runs a side, so 10,000 items at most; for a larger dataset, score a fixed random sample of it and say so.

**Stored cases:** To score a version on the decision's labeled cases (the improve flow), send `cases` in place of `items`: `{evaluation_ids}` from `list_cases`, or its filters (`outcomes`, `structure_hash`, `label_source`, `since`, `until`) with `limit` and `cursor` (`cases.next_cursor` back for the next page). Always dry runs. Evaluation ids named one by one are each scored, even near-identical ones; filters get one case per group, as `list_cases` lists them. Each case reuses the answers it stored for questions the version didn't change, so only new or reworded questions spend; `refresh` names questions to ask afresh. The run keeps each case's label, label source and answer corrections. A preview is exactly two such runs and a `loss_report`: the base with `refresh` set to the base questions the candidate changed or dropped, and the candidate, both with `cases.outcomes` set to the outcomes either version can produce and paged over the same cases (a preview judges the newest 500). Pages aren't a snapshot: a label given while you page can move a case between pages. It can differ on a case only one version can read.

**Gotchas:** Loss is how wrong a version is, weighted by the objective's severities and by outcome so each expected outcome counts equally; `cost` in a run's response is spend (`model_calls`, `questions`, tokens and `usd_micros`). Never pass the results back yourself: pass `run_ids`, so they don't go through your context. `get_run` reads a run's response back until it expires, as that call answered it (a run made with `results: "none"` reads back with none), so use it only to recover a run you lost, never to judge one. An item whose model calls fail is that item's result (`status: "failed"`, `error.retryable`): `loss_report` leaves retryable failures out of both sides (`excluded.failed_transient`) and prices the rest as mistakes, so resend the retryable ones. Under 80% of the labeled items scored, the candidate isn't accepted until you do. Runs expire after a day. Price base and candidate on the same items: items the base lacks are counted in `excluded.no_base`. Each item that succeeded is billed as a dry run; a failed one's model spend is still recorded.

**What to report:** each side's full report (`detail: "full"`) carries the standard classification metrics beside the loss: `macro_f1` (mean F1 across outcomes), `accuracy` (unweighted, so the commonest outcome dominates it), `outcomes` (each outcome's precision, recall and F1, and how often it was expected and given), `confusion` (every expected → given cell with a case, with its count, right answers included) and `by_mistake` (every wrong cell with its count and loss, the most loss first), each side's worst or largest 1,000 cells at most, kept on both sides and in both lists (a few thousand at most), with `confusion_total` and `by_mistake_total` counting them all. Report them to the person, base beside candidate. Loss decides what is kept, because it is the objective they set; the metrics help them see what changed.

**Done when:** `loss_report` shows the candidate's loss per case against the base's, with `accepted`, and you have looked at the worst cases it lists.

## List labeled cases

**Goal:** See the labels a decision has: which cases it gets wrong now, who labeled them, and what reviewers said.

**Find:** “Reads only. The decision's labeled cases in the key's environment” (`list_cases`). Read its `inputSchema` first.

**Request fields:** Path `decision_id`; query `limit` (default 50, at most 200, or 50 with `include=content`), `cursor`, `evaluation_ids` (comma-separated `evl_` ids, at most 200, or 50 with `include=content`: just those cases, in that order, in one page, with no `limit` or `cursor`; the ids `loss_report` rows carry over case runs), and filters: `outcomes` (comma-separated without spaces, or a JSON array of strings when a name has a comma), `structure_hash` (the version a case ran under), `since` and `until` on when the evaluation ran, and `label_source` (`human`, `synthetic`, `unknown`). `structure_hash`, `since` and `until` choose which evaluations count before one case per group is kept, so a case labeled under an older version still lists under that version's hash, and `excluded` counts only those evaluations. Needs `decisions:develop`.

**Result:** `items` (the cases), newest label first, each with `evaluation_id`, `item_id`, `expected` (the label), `label_source`, `consensus` (`owner`, `majority`, `example`), `feedback_source`, `split`, `ran` (`version: {structure_hash, number}`, `model: {profile}`, and the `status` and `outcome` it gave then), `answer_corrections` (`{question_hash, questions, expected}`), `reasons` (what people wrote), `feedback_ids`, `created_at`, `labeled_at`, and `content` only with `include=content`. Also `total` (matching cases), `unmatched` (with `evaluation_ids`: ids that aren't a labeled case here, or that the filters dropped), `excluded` (evaluations with feedback that are not cases, by reason), `next_cursor` and `has_more`.

**Gotchas:** These are the cases a preview replays: one per group (the same item, or near-identical items with the same label; a boundary pair whose labels differ stays two), with the same exclusions; set `outcomes` to the outcomes your versions produce, and a preview replays the newest 500 of that list. To score a version on them, send their `evaluation_id`s (or the same filters) to `create_run` with `cases` ([Score a dataset](#score-a-dataset)). An imported example's `label_source` is the one it was imported with (`unknown` when none was sent). A case whose `ran.outcome` differs from `expected` is one the version it ran under got wrong. A cursor works only with the filters it was issued for. Content stays on the server unless asked for; reasons are often the best clue for a rewrite.

**Done when:** You have the cases you need, with the counts in `excluded` explaining what is missing.

## Set the objective

**Goal:** Store what the decision optimizes for, so previews and optimizers score against the same objective as the decision file.

**Find:** Find “Set what a decision optimizes for” and “Get what a decision optimizes for” (`set_objective`, `get_objective`; `PUT` and `GET /v1/decisions/{decision_id}/objective`). Read each `inputSchema` before first use.

**Request fields:** The objective lives in the decision file beside the structure: `{decision_id, structure, objective}`. Write the person's answers as `mistakes` (how to ask: [the build flow's `objective` stage](build.md#objective)) (`{expected, got, severity}`, with severity `minor`, `normal`, `costly` or `severe`, or a direct `loss`); add `fallback`, the outcome whose mistakes are cheapest (sending a case there wrongly counts as `minor` unless listed; it routes nothing), and, only when the person wants one, a `target` as `{"max_loss_per_case": <number>}`: a soft reference point that reports mark as met or not (`target_met`), never a reason to stop. There are no limits on an outcome's share, recall or precision (an objective with `limits` is refused): scores weigh each expected outcome equally however many cases it has, so an example set's mix doesn't change them. A two-outcome decision can use `{"outcome": "<one>", "priority": "catch_all" | "be_sure" | "balanced"}`. Send `{"objective": <the file's objective>}`; `{"objective": null}` clears it. `set_objective` always answers 200 with `changed`: false when the stored objective already matched, so no new revision was written.

**Gotchas:** The objective never changes `structure_hash`. Each environment keeps its own; push it with the key you preview with. 404 until the decision exists. Read `unknown_outcomes` in the response: an outcome the newest structure can't produce is left out when scoring, usually a typo or a renamed outcome. Without an objective the decision is scored with balanced losses. Previews and `loss_report` judge base and candidate by it.

**Done when:** `get_objective` shows the hash of the decision file's objective, and `unknown_outcomes` is empty. To check, compare the `objective` `get_objective` returns with the decision file's, field by field; don't hash it by hand.

## Check cost

**Goal:** Read evaluation and preview spending for a decision and window in the intended environment.

**Find:** Find “Get a decision's model cost, per evaluation and in total” (`get_usage`). Read its `inputSchema` first and use its decision/window selectors.

**Response fields:** `environment`, `window`, `evaluations[]` grouped by `model.profile` and `assignment_mode` (`primary` or `example`), with `model_calls`, token counts, and `usd_micros: {total, mean, p50, p95}`; also `models[]` (rows of `resolved_model`, `adapter`, `calls`, and the tokens and cost) and `previews: {runs, questions, input_tokens, usd_micros}`. Amounts are in micro-dollars. Each group also carries `estimated_usd_micros` (the estimated part), and the report adds `other_spend[]` (spend outside evaluations and previews, such as attempts that never committed) and a `total`.

**Per-request cost:** Evaluate (including dry runs), `get_evaluation`, runs, `import_examples` and previews report one `cost` shape: `{model_calls, questions, cache_hits?, input_tokens, output_tokens, usd_micros}`, and on evaluate also `calls`. `model_calls` counts upstream model calls; one call can answer several questions (Jev compiles them). `questions` counts the questions sent to a model. `cache_hits` is present on previews and case runs, and on evaluate only when the cache served a question. `get_evaluation` omits `cost` if no usage row exists. `usd_micros` is millionths of a US dollar. Per-request `cost` is reported cost only.

**Scale and limits:** Costs are small: a few items cost a fraction of a cent on the default model. The report breaks out estimates for abandoned calls and lists evaluations that never committed under `other_spend`; `total` adds everything recorded.

**Done when:** You have recorded the environment, window, evaluation and preview costs, the other spend, and how much of the total is estimated.

## Wire it into the app

**Goal:** Have the person's app call the decision in production, once a version is kept.

**What the app needs:**

- **A key.** Signing in serves the agent, not a backend: the app needs an API key with `decisions:evaluate` for the environment it runs in (`env:test` for a test key). Only a team admin can create one, in Moonbounce's app under **Team → Keys**; a member asks an admin. The app stores it as a secret, never in code.
- **One call per item:** `POST https://api.moonbounce.ai/v1/evaluate` with `Authorization: Bearer <key>`, `Content-Type: application/json` and an `Idempotency-Key` header, and a body of `state` (what the questions read), `attribute_values` (only for attributes the structure doesn't derive with `from`; when a new version derives one the app used to send, the app stops sending it in the same release), and `moonbounce: {decision_id, item_id, structure_hash}`. Sending the registered hash keeps the structure in the decision file; the structure inline works too and registers it on first sight. `item_id` is the app's own id for the item: feedback on it joins up later.
- **Reading the result:** check `status` first. `decided` carries `outcome`, `computed` carries `value`, and `abstained` carries at most the structure's `abstain` outcome. When `missing` lists questions, the result rests on partial evidence; decide with the person what the app does then (a person's review, say).
- **Retries,** always with the same `Idempotency-Key`: a 503 (transient), after its `Retry-After` when it sends one, with backoff otherwise; a 429 `RATE_LIMITED`, after its `Retry-After`. Don't retry a 429 `QUOTA_EXCEEDED` before `details.reset_at` (the daily quota is spent), a 502 `ENGINE_FAILED` (the model failed on this input, and retrying spends again), or a 400 (a bug in the request). Decide with the person what the app does with an item it can't evaluate. A single call's budget is 10 seconds.

**Stop** before changing the app: show the person the call, where it would go in their code, and how the app handles each status, and write it only when they say yes. There is no published SDK yet, so the call is plain HTTP in the app's language.

**Done when:** the person has approved the change, and one real call from the app returns the expected `environment`, `version.structure_hash` and status.

## Retries for every write

Every write except `fill_review_queue` takes an `idempotency-key` argument. The same key and request replay the first result for 24 hours; the same key with a different request returns 422 `IDEMPOTENCY_KEY_REUSED`. Keys are scoped to organization plus environment across every operation. Send a key on each write and reuse it only for retries of that request. Done when a successful result is recorded under that request's key.

Fresh evaluate and `import_examples` responses always include `cost`. Single-item dry runs carry cost because they cannot replay. A replay returns the first response as it was stored, so for 24 hours after a release that changed a response's shape, a replay of a call made before it comes back in the old shape. A replay's cost is the original spend, not a second charge.

## Read back or list

**Goal:** Find existing decisions and inspect what was registered or evaluated.

**Find:** `list_decisions`, `get_decision`, `list_versions`, `get_version`, `list_evaluations` and `get_evaluation` have descriptions saying “Reads only.” Read the discovered schemas first.

**Request:** `list_decisions`, `get_decision`, `list_versions` and `get_version` accept either `decisions:evaluate` or `decisions:develop`; evaluation reads require `decisions:evaluate`. Lists accept `limit` (1–100, default 50) and `cursor`. Follow `next_cursor` while `has_more` is true. Evaluation filters include `item_id`, `assignment_mode` and inclusive `since` / exclusive `until`.

**Gotchas:** Decisions and versions belong to the whole organization: test and production keys list the same ones, `created_at` is when they were first registered, and a decision registered in production can be listed by a test key before it has a model setting there. Evaluations are environment-specific; lists omit content. `list_decisions` and `get_decision` items carry `version_count` and the `newest_version`; `list_versions` items carry each version's `structure_hash`, `number` and question hashes in `question_hashes`, newest number first; `list_evaluations` items carry `status`, `value`, `outcome` and `version.structure_hash`. `register_decision` returns hashes without the structure; `get_version` takes the hash or the number as `ref` and returns the structure as `structure`, in authored key order, with its hashes. `get_evaluation` returns the evaluate shape plus `assignment_mode`, the recorded `cost` when a usage row exists, and `content: {state, attribute_values?}` when stored, otherwise `content: null`. A stored JSON null is `{state: null}` inside `content`. The internal `store_state` setting defaults to true. `list_feedback` remains HTTP only.

**Done when:** The requested page or record is read in the intended environment; fetch a definition or state only when the task needs it.

## Dry run

**Goal:** Run an item without creating an evaluation or feedback target.

**Find:** `evaluate`, with `moonbounce.dry_run: true`.

**Request:** Use an inline structure, or a hash in `moonbounce.structure_hash`: a registered one, or one a live upload or run of the decision holds in this environment, without registering it ([Upload a file](#upload-a-file)). A single-item dry run accepts an `idempotency-key` and ignores it, so a retried dry run runs, and is billed, again. A run (`create_run`) takes one and replays: retrying the same run after a lost response reuses its key and replays the first result, while a new run of just the failed items takes a new key.

**Gotchas:** This still spends model calls and uses ordinary admission limits. It stores only its usage, a `dry_run` billable unit and, if the org has none yet, the org's row: no registration, model setting, content key or answer cache is created, so `version.number` is null for a structure nobody registered. Usage totals appear under `unknown` profile/mode without a per-model breakdown. Repeating the call can spend again. Reading or submitting feedback about a `dry_` id is refused; MCP may reject its pattern before calling the API. Use a normal evaluation when the result should be kept.

**Done when:** The response has `dry_run: true`, a `dry_` id, the intended `environment`, `cost` and the result; keep it locally if needed.
