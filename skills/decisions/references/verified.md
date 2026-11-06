# Verified lessons and limits

Evidence convention: **observed** means run live against `https://app.moonbounce.ai` with a test-environment key on 2026-10-27, on a throwaway decision; **observed locally** means run against a local API (`bun run dev` in decisions-platform) with a fake Test key on the date given; **from docs** means read only (Moonbounce's guides, tool descriptions, or planned API changes). Re-run discovery before relying on dated behavior.

## A. Lessons checked live (test key, 2026-10-27)

The entries distinguish confirmed checks from lessons the live check could not establish.

**from docs** — These checks ran before the System One shape and have not been rechecked live on it; each lesson is restated in today's terms ([tasks.md](tasks.md)) and kept only where it still holds in substance.

1. **Backtest does not register the candidate — confirmed.** **observed** — After previewing a candidate, evaluating its structure hash returned 404 `VERSION_UNKNOWN`: that the structure is not registered and should be sent inline or registered first. **from docs** — The current reason is `STRUCTURE_UNKNOWN`; the preview tool's description also says to register first. Register the candidate or evaluate it inline first. Planned preview registration would make this lesson stale; see section D.

2. **Imported examples are separate from primary traffic — confirmed.** **observed**, under the earlier name `import_seeds` — Feedback by item on an item that had only imported examples returned 404, saying feedback by item binds to primary traffic and naming `about.evaluation_id`; feedback by the evaluation id from the import's response worked. **from docs** — With `import_examples` the 404 says the item has only imported examples and names the example's evaluation id; the response carries that id as each result's `id`.

3. **Backtests list only changed items — confirmed.** **observed** — `changed_items` holds only fixes and breaks. Unchanged cases and the value distribution are unavailable, so keep cut-point tuning separate from this response.

4. **`agreed` mixes outcome and answer agreement — superseded.** **observed**, under the earlier `import_seeds` — An example with a matching outcome label but a contradictory per-question label returned `agreed:false` with the disagreement in `conflicts`. **from docs** — `import_examples` returns the run's response, with no `agreed`, `conflicts` or `total`: agreement is `loss_report({candidate: {run_ids: [run_id]}})`, which judges each example's `expected_outcome` label. Not yet checked live.

5. **Keep a test set written before designing against labels — advice, not a testable result.** **from docs** — Only owner, reviewer, and imported-example judgments count, and only `human` labels count toward the no-new-severe-mistake rule; evidence labels are Imported examples (`imported_examples`) and Training cases, both in-sample (the API lists no Validated label today). Keep examples used to design a change separate from evidence used to judge it.

6. **Costs were tiny in this check — confirmed.** **observed** — The whole check cost under USD 0.001; ten imported examples on Jev cost about USD 0.00017. See [Check cost](tasks.md#check-cost) for the recorded workload and accounting limits.

7. **The end-to-end flow — confirmed.** **observed** — Structure → register or evaluate inline → labeled data → backtest → ship by evaluating with the new structure worked, with the registration caveat in lesson 1.

8. **Scoring a dataset by run — confirmed.** **observed locally (2026-11-05)**, under the earlier names `evaluate_batch`, `batch_id` and `cost_report` (now `evaluate` with `items`, `run_id` and `loss_report`) — a run with an inline structure and three labeled items returned its id, `counts` (`labeled: 3`), `cost` (money) and the `version` it ran (`structure_hash`, `engine_profile`, `binding`); three items on Jev cost USD 0.000191. The report on that id priced them without the results passing back, with `objective.source: "balanced"` since no objective was stored. Over MCP both tools take their request under `body`; sending the request's fields at the top level of the tool arguments returned 400 `REQUEST_INVALID` (“Malformed JSON in request body”).

## B. Docs versus live tools

- **observed** — No contradictions were found in the claims checked. The 25-tool list matched OpenAPI operations minus two HTTP-only operations (health and feedback export), plus one hand-written tool.
- **observed** — The public OpenAPI document at `https://app.moonbounce.ai/openapi.json` was byte-identical after key sorting to the reference API schema.
- **observed** — Moonbounce's engine guide's binding example, `created:true`, and `version.binding` shapes matched live results.
- **observed** — JSON-RPC `tools/list` supplied all tool schemas inline. No whoami tool was present. Binding, preview, and usage reads exposed environment; evaluate responses did not. A missing binding reported “in test.”
- **observed locally (2026-11-05)** — `tools/list` returned 37 tools, `whoami` among them (org, `environment`, key id, actor, scopes), along with `evaluate` with `items`, `loss_report`, `get_objective` and `set_objective`.
- **from docs** — Keys remain bound to one environment: production, or test with `env:test`. Test traffic never counts as production, and its data and previews are separate. Production-key behavior was outside the live check.

## C. Could not verify

- **from docs** — A test key's preview can fall back to production's binding before having its own, according to Moonbounce's engines and environments guide; verifying that needs production data.
- **from docs** — Review-queue tools serve human reviewers; that flow was not tested.
- **from docs** — Production-key behavior was not tested.
- **from docs** — Enforcement of `decisions:develop` and `decisions:feedback` beyond the test key's success was not tested. **observed** — The test key's owner feedback succeeded.
- **observed** — API operations were exercised through MCP only; REST-only paths were not exercised. OpenAPI equivalence was checked by diff.
- **observed** — The engine listing marked image modality “preview”; image behavior was not tested.

## D. Changing soon (as of 2026-10-27)

Re-check each item against the current tools and schemas before relying on it.

- **from docs** — Approved API changes include whoami and `environment` on every response; previews that register candidates, making lesson 1 stale; per-case preview scores, enabling cut-point tuning; a `dry_run` option and list calls for decisions/structures/evaluations. Whoami, dry runs, the list calls, the decision objective and the loss report (on previews, and on its own over `evaluate` with `items` results) are implemented; whoami and run scoring were checked locally (lesson 8, section B), the rest have not been rechecked live. See the task reference.
- **from docs** — Implemented, not yet checked live: the flows ([build.md](build.md), [improve.md](improve.md)) and the `build_decision` and `improve_decision` MCP prompts; `evaluate` with `cases` takes the evaluation ids `list_cases` gives as stored items, and `import_examples` (earlier `import_seeds`) imports labeled examples as a stored run with each label's `label_source`, with per-example failures and the run's response; `submit_feedback` takes many labels as `submissions`.

MCP tools use snake_case operation names. The binding, preview, import, usage, and review-shown tools are `set_binding`, `preview`, `import_examples`, `get_usage`, and `mark_review_shown`. `tasks.md` uses the current names; dated observations above describe the behavior checked at the time.
