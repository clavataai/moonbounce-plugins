# The refine loop

Both flows end in this loop. Flow 1 ([build.md](build.md)) runs it over the repo's dataset, and flow 2 ([improve.md](improve.md)) runs it over the decision's labeled cases. Each pass scores a candidate under the decision's objective, reads the loss report to see where the loss sits, edits the structure and scores again. A change is kept only when the loss report accepts it. The loop stops when the person says the decision is good enough or the objective's `target` is met.

## Roles

The **author** edits the structure, and you (the orchestrator) score it. Scoring is mechanical, so you can do it yourself. The author never scores its own version and never reads a set it hasn't been given:

- In flow 1, the author is given the repo's dataset and never the blind set.
- In flow 2, the author is given the decision's cases.

Give the author the report's roll-ups and the worst cases it lists, and give it each case's `reason` from the dataset or `list_cases`. Reasons are often the best clue for a rewrite.

## Score a version

Never score by hand, and never pass the results back yourself. Pass `run_id`s so the results stay out of your context. Over MCP, `evaluate` and `loss_report` take their request under `body`. [Score a dataset](tasks.md#score-a-dataset) has the full request shapes.

- **A dataset (flow 1, and flow 2's synthetic cases).**
  - Call `evaluate` with `items` and `moonbounce.dry_run: true`. Always use a dry run: a stored run would add evaluations of an unshipped version to the decision.
  - Send the version's structure fields at the top level, or `moonbounce.structure_hash`, plus `moonbounce.decision_id`.
  - Send up to 200 items per call, each as `{item_id, state, attribute_values?, expected, label_source}`.
  - Turn the repo's JSONL into items with:

    ```sh
    jq -c '{item_id, state} + (if .attribute_values then {attribute_values} else {} end) + {expected: ((.claims // [])[] | select(.kind == "expected_outcome") | .value), label_source: (.label_source // "unknown")}' moonbounce/<decision_id>.examples.jsonl
    ```

    A line with no `expected_outcome` drops out of the output: it isn't labeled yet, or it labels answers only, so it isn't scored by outcome.
- **Stored cases (flow 2).**
  - Call `evaluate` with `cases` in place of `items`: either `{evaluation_ids}` from `list_cases`, or its filters with `limit` and `cursor`. Cases are always dry runs.
  - Each case reuses the answers it stored for questions the version didn't change.
  - Filter by `label_source` to score human-labeled cases apart from the rest: real labels decide in flow 2.
  - A `preview` is two of these runs and a `loss_report` in one call, over the newest 500 cases. It returns compact reports with no `run_id`s to page and no split by label source, so flow 2 scores with the runs themselves.
- **Judge.**
  - Call `loss_report` with `{candidate: {run_ids}, base: {run_ids}}`. It returns `loss_per_case`, `accepted` and `reasons`, plus the roll-ups below.
  - Page through the cases behind a roll-up with `cases: {filter: {kind: "wrong"}}`. The other kinds are `severe`, `changed`, `missing_criterion`, `{kind: "cell", expected, got?}`, `{kind: "question", question}` and `{kind: "near_cut", at}`.
  - Price base and candidate on the same items. Items the base lacks are counted in `excluded.no_base` and left out, so when the dataset grows, score the base on the new items too and add that run to its `run_ids`.
  - Runs expire after about a day. Keep the run ids with their structure hash and expiry (flow 1 keeps them in `work/runs.json`), and re-score the last kept version when its runs have expired.
- **Failures.**
  - An item whose engines fail permanently (unreadable content, an engine refusal) is counted as its dearest mistake.
  - A retryable failure (`NOT_STARTED`, `ENGINE_UNAVAILABLE`) is left out as `excluded.failed_transient`. Resend those items in a new run with a new Idempotency-Key, and list both runs. Runs expire after a day.
- **Same objective.** Store the objective (`set_objective`) before the first scoring. Without one, `loss_report` falls back to balanced losses. Compare only reports with the same `objective.effective_hash`.

## Keep or drop a change

Keep a candidate when both of these hold:

- **The report accepts it.** `accepted` is true, with the last kept version as the base: lower loss per case, and no new severe mistake on a human-labeled case.
- **It adds no severe mistake on any label.** The no-new-severe-mistake rule counts only `human` labels, so when most labels are synthetic, also page the severe cases (`cases: {filter: {kind: "severe"}}`). Drop the candidate when any row has `base.severe: false`: a case the base got right is now one of the objective's severe mistakes, whatever its label.

The first structure that validates and scores is the first kept version: it is the base the next candidate must beat. `loss_report` never accepts a candidate without a base, so don't wait for the target to keep a first version. The target is when to stop, not when to keep.

Register each kept version (`register_structure`), so its hash works as a base and at import, and commit it: the decision file, with its structure hash and loss per case in the commit message. The next candidate starts from the last kept version, or from a dropped one when the author judges that one closer.

Engines answer the same case differently from run to run. When a verdict hinges on one or two cases, score both versions twice. Small sets are only indicative.

## From evidence to a change

Read the report top down. Each roll-up points at one kind of change.

| The report shows | Change | Spends engine calls |
| --- | --- | --- |
| `near_cuts`: wrong cases just below or above a cut whose expected outcome is the next band (`lower_it`, `raise_it`) | Move that cut in the direction that fixes more than it breaks | No (numbers) |
| `by_mistake`: the loss sits in one cell (`expected` → `got`) between two outcomes, with no single suspect question | Re-weight the inputs of the aggregation that separates those outcomes, or reorder its rules | No (numbers) |
| `by_question`: one question carries a large share of the loss | Rewrite that question's instructions or criteria from the cases' reasons; keep its name if its meaning stays | Yes (the rewritten question) |
| `missing_criterion`: wrong cases with no suspect, clustered on one theme | Add a question for that criterion and an input for it in the aggregation | Yes (the new question) |
| `answer_conflicts` on a case: a reviewer's `expected_answer` disagrees with the answer | Rewrite that question, as for `by_question` | Yes |
| `failed`: permanent failures | Check `reads` against the inputs the app always sends, and read an optional field through its parent | No |
| A label that contradicts the brief | Fix the label or the brief, not the structure: go back to `labels` (flow 1) or `brief_check` (flow 2) | No |

**Numbers come before questions.** Cut points and weights cost nothing to change. A question change costs engine calls on every case. So try the numbers the report points at before rewriting a question, and change one thing per candidate so the report can tell you what it did.

How much a re-score costs depends on the flow:

- **Flow 1.** A dry run over `items` asks every question again, so batch the number changes the report supports into one candidate.
- **Flow 2.** A run over `cases` reuses the stored answers for questions the candidate didn't change, so a numbers-only candidate makes no engine calls (`cost.engine_calls` stays 0).

**Cost.** Every run's response carries `cost` (money; `usd_micros` is millionths of a dollar). Agree a budget with the person before the loop. `get_usage` reports the decision's spend over a window, including dry runs.

## When to stop

Stop when the person says the decision is good enough, or when `target_met` is true. When three candidates in a row are dropped, stop and report to the person what the report shows. The brief or the labels are usually the problem, not the structure.
