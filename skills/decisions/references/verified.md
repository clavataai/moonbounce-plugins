# Verified lessons and limits

Evidence convention: **observed** means run live against `https://app.moonbounce.ai` with a test-environment key on 2026-10-27, on a throwaway decision; **from docs** means read only (Moonbounce's guides, tool descriptions, or planned API changes). Re-run discovery before relying on dated behavior.

## A. Lessons checked live (test key, 2026-10-27)

The entries distinguish confirmed checks from lessons the live check could not establish.

1. **Backtest does not register the candidate — confirmed.** **observed** — After previewing a candidate, evaluating its structure hash returned 404 `VERSION_UNKNOWN`: “structure sha256:... is not registered ...; send the spec inline or push it first.” **from docs** — The preview tool's description also says this. Register the candidate or evaluate it inline first. Planned preview registration would make this lesson stale; see section D.

2. **Seeded examples are separate from primary traffic — confirmed.** **observed** — Feedback by item returned 404: “Item s1 in <decision> has only seed evaluations; feedback by item binds to primary traffic. Address the seed by about.evaluation_id (evl_...).” Feedback by the evaluation id from the seed response worked. The error now names the fix.

3. **Backtests burn held-out cases and list only changed items — confirmed.** **observed** — The one held-out case listed by the backtest subsequently read back as `split:"burned"` through the feedback read tool. `changed_items` holds only fixes and breaks. Unchanged cases and the score distribution are unavailable, so keep cut-point tuning separate from this response.

4. **Combine behavior — confirmed.** **observed** — Adding a third term to a two-term mean combine moved a refund-plus-angry message from 0.73 to 0.497, flipping escalate to approve: a break in the preview. In another comparison the same message scored 0.64 with mean and 0.998 with `prior:"any"`. A `lowers` term with a positive fixed weight returned 400, “every term needs a fixed weight whose sign matches its effect,” with `param` set to `spec.nodes.<node>.fixed.weights.<feature>`. Sign matching is enforced by the server.

5. **`aligned` (now `agreed`) mixes outcome and feature agreement — confirmed.** **observed** — A seed with a matching outcome label but a contradictory feature claim returned `agreed:false`, `conflicts:[{feature,expected,got}]`, and `aligned:0`. `aligned` counts examples with neither kind of conflict; inspect `conflicts` to identify feature disagreements. **from docs** — Moonbounce's feedback guide describes `agreed` as agreement with the label and `conflicts` as feature disagreements, but omits that a feature conflict alone makes `agreed` false.

6. **“The generator only proposed a rewrite and a hand-designed feature did better” — not verified.** **observed** — Generation returned null on the small live set of 11 seeds and two reviews. **from docs** — The tool description permits “a feature rewrite or a new feature.” Treat a suggestion as a lead and compare it against your own candidate with a backtest.

7. **Keep a test set written before designing against labels — advice, not a testable result.** **from docs** — Only owner, reviewer, and seed judgments count; evidence labels distinguish Starting examples/Training cases from Held out (the API lists no Validated label today). Keep examples used to design a change separate from evidence used to judge it.

8. **Costs were tiny in this check — confirmed.** **observed** — The whole check cost under USD 0.001; ten seeded examples on Jev cost about USD 0.00017. See [Check cost](tasks.md#check-cost) for the recorded workload and accounting limits.

9. **The end-to-end flow — confirmed.** **observed** — Spec → register or evaluate inline → labeled data → backtest → ship by evaluating with the new spec worked, with the registration caveat in lesson 1.

## B. Docs versus live tools

- **observed** — No contradictions were found in the claims checked. The 25-tool list matched OpenAPI operations minus two HTTP-only operations (health and feedback export), plus the hand-written apply-suggestion tool.
- **observed** — The public OpenAPI document at `https://app.moonbounce.ai/openapi.json` was byte-identical after key sorting to the reference API schema.
- **observed** — Moonbounce's engine guide's binding example, `created:true`, and `version.binding` shapes matched live results.
- **from docs** — The feedback page's `aligned`/`agreed` explanation is incomplete as described in lesson 5. Fixed-weight sign enforcement is stated in the tool schema error; fixed-weight signs must match each term’s effect. **observed** — The server enforced the mismatch with 400.
- **observed** — JSON-RPC `tools/list` supplied all tool schemas inline. No whoami tool was present. Binding, preview, and usage reads exposed environment; evaluate responses did not. A missing binding reported “in test.”
- **from docs** — Keys remain bound to one environment: production, or test with `env:test`. Test traffic never counts as production, and its data, previews, and suggestions are separate. Production-key behavior was outside the live check.

## C. Could not verify

- **from docs** — A test key's preview can fall back to production's binding before having its own, according to Moonbounce's engines and environments guide; verifying that needs production data.
- **observed** — Suggestion generation on a real set remains unverified; the small checked set returned null.
- **from docs** — Review-queue tools serve human reviewers; that flow was not tested.
- **from docs** — Production-key behavior and suggestion acceptance were not tested. Acceptance is described as production-only.
- **from docs** — Enforcement of `decisions:develop` and `decisions:feedback` beyond the test key's success was not tested. **observed** — The test key's owner feedback succeeded.
- **observed** — API operations were exercised through MCP only; REST-only paths were not exercised. OpenAPI equivalence was checked by diff.
- **from docs** — Weights fitting was not verified; none exists yet. **observed** — Default weights reported `weights_fitted:false`.
- **observed** — The engine listing marked image modality “preview”; image behavior was not tested.

## D. Changing soon (as of 2026-10-27)

Re-check each item against the current tools and schemas before relying on it.

- **from docs** — Approved API changes include whoami and `environment` on every response; previews that preserve held-out cases by default; previews that register candidates, making lesson 1 stale; per-case preview scores, enabling cut-point tuning; a `dry_run` option and list calls for decisions/structures/evaluations. The dry-run and list contracts are now implemented but have not been rechecked live; see the task reference.

MCP tools use snake_case operation names. The binding, preview, seeding, usage, and review-shown tools are `set_binding`, `preview`, `import_seeds`, `get_usage`, and `mark_review_shown`. `tasks.md` uses the current names; dated observations above describe the behavior checked at the time.
