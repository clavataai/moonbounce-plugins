# Verified lessons and limits

Evidence convention: **observed** means run live against `https://app.moonbounce.ai` with a test-environment key on 2026-10-27, on a throwaway decision; **observed locally** means run against a local API (`bun run dev` in decisions-platform) with a fake Test key on the date given; **from docs** means read only (Moonbounce's guides, tool descriptions, or planned API changes). Re-run discovery before relying on dated behavior.

## A. Lessons checked live (test key, 2026-10-27)

The entries distinguish confirmed checks from lessons the live check could not establish.

**from docs** — These checks ran before the System One shape and have not been rechecked live on it; each lesson is restated in today's terms ([tasks.md](tasks.md)) and kept only where it still holds in substance.

1. **Backtest does not register the candidate — confirmed.** **observed** — After previewing a candidate, evaluating its structure hash returned 404 `VERSION_UNKNOWN`: that the structure is not registered and should be sent inline or registered first. **from docs** — The current reason is `STRUCTURE_UNKNOWN`; the preview tool's description also says to register first. Register the candidate or evaluate it inline first. Planned preview registration would make this lesson stale; see section D.

2. **Seeded examples are separate from primary traffic — confirmed.** **observed** — Feedback by item returned 404: “Item s1 in <decision> has only seed evaluations; feedback by item binds to primary traffic. Address the seed by about.evaluation_id (evl_...).” Feedback by the evaluation id from the seed response worked. The error now names the fix.

3. **Backtests list only changed items — confirmed.** **observed** — `changed_items` holds only fixes and breaks. Unchanged cases and the value distribution are unavailable, so keep cut-point tuning separate from this response.

4. **`agreed` mixes outcome and answer agreement — confirmed.** **observed** — A seed with a matching outcome label but a contradictory per-question label returned `agreed:false` and the disagreement in `conflicts` (now `{question, expected, got}`). The batch's `agreed` count includes only examples with neither kind of conflict; inspect `conflicts` to identify question disagreements.

5. **“The generator only proposed a rewrite and a hand-designed feature did better” — not verified.** **observed** — Generation returned null on the small live set of 11 seeds and two reviews. **from docs** — The tool description permits “a question rewrite or a new question.” Treat a suggestion as a lead and compare it against your own candidate with a backtest.

6. **Keep a test set written before designing against labels — advice, not a testable result.** **from docs** — Only owner, reviewer, and seed judgments count; evidence labels are Starting examples and Training cases, both in-sample (the API lists no Validated label today). Keep examples used to design a change separate from evidence used to judge it.

7. **Costs were tiny in this check — confirmed.** **observed** — The whole check cost under USD 0.001; ten seeded examples on Jev cost about USD 0.00017. See [Check cost](tasks.md#check-cost) for the recorded workload and accounting limits.

8. **The end-to-end flow — confirmed.** **observed** — Structure → register or evaluate inline → labeled data → backtest → ship by evaluating with the new structure worked, with the registration caveat in lesson 1.

9. **Scoring a dataset by batch — confirmed.** **observed locally (2026-11-05)** — `evaluate_batch` with an inline structure and three labeled items returned a `batch_id`, `counts` (`labeled: 3`), `cost` and the `version` it ran (`structure_hash`, `engine_profile`, `binding`); three items on Jev cost USD 0.000191. `cost_report` with that `batch_id` priced them without the results passing back, by balanced costs (`objective.source: "balanced"`) since no objective was stored. Over MCP both tools take their request under `body`; sending `structure` and `items` at the top level returned 400 `REQUEST_INVALID` (“Malformed JSON in request body”).

## B. Docs versus live tools

- **observed** — No contradictions were found in the claims checked. The 25-tool list matched OpenAPI operations minus two HTTP-only operations (health and feedback export), plus the hand-written apply-suggestion tool.
- **observed** — The public OpenAPI document at `https://app.moonbounce.ai/openapi.json` was byte-identical after key sorting to the reference API schema.
- **observed** — Moonbounce's engine guide's binding example, `created:true`, and `version.binding` shapes matched live results.
- **from docs** — The feedback page's `agreed` explanation is incomplete as described in lesson 4.
- **observed** — JSON-RPC `tools/list` supplied all tool schemas inline. No whoami tool was present. Binding, preview, and usage reads exposed environment; evaluate responses did not. A missing binding reported “in test.”
- **observed locally (2026-11-05)** — `tools/list` returned 37 tools, `whoami` among them (org, `environment`, key id, actor, scopes), along with `evaluate_batch`, `cost_report`, `get_objective` and `set_objective`.
- **from docs** — Keys remain bound to one environment: production, or test with `env:test`. Test traffic never counts as production, and its data, previews, and suggestions are separate. Production-key behavior was outside the live check.

## C. Could not verify

- **from docs** — A test key's preview can fall back to production's binding before having its own, according to Moonbounce's engines and environments guide; verifying that needs production data.
- **observed** — Suggestion generation on a real set remains unverified; the small checked set returned null.
- **from docs** — Review-queue tools serve human reviewers; that flow was not tested.
- **from docs** — Production-key behavior and suggestion acceptance were not tested. Acceptance is described as production-only.
- **from docs** — Enforcement of `decisions:develop` and `decisions:feedback` beyond the test key's success was not tested. **observed** — The test key's owner feedback succeeded.
- **observed** — API operations were exercised through MCP only; REST-only paths were not exercised. OpenAPI equivalence was checked by diff.
- **from docs** — Weights fitting was not verified; none exists yet.
- **observed** — The engine listing marked image modality “preview”; image behavior was not tested.

## D. Changing soon (as of 2026-10-27)

Re-check each item against the current tools and schemas before relying on it.

- **from docs** — Approved API changes include whoami and `environment` on every response; previews that register candidates, making lesson 1 stale; per-case preview scores, enabling cut-point tuning; a `dry_run` option and list calls for decisions/structures/evaluations. Whoami, dry runs, the list calls, the decision objective and the cost report (on previews, and on its own over `evaluate_batch` results) are implemented; whoami and batch scoring were checked locally (lesson 9, section B), the rest have not been rechecked live. See the task reference.
- **from docs** — Planned: a case listing whose evaluation ids `evaluate_batch` can take as stored items, and `import_examples` in place of `import_seeds`, carrying each label's source.

MCP tools use snake_case operation names. The binding, preview, seeding, usage, and review-shown tools are `set_binding`, `preview`, `import_seeds`, `get_usage`, and `mark_review_shown`. `tasks.md` uses the current names; dated observations above describe the behavior checked at the time.
