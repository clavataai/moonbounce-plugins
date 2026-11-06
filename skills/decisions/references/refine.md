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
- **Same objective.** Store the objective (`set_objective`) before the first scoring. A new decision exists only once a structure is registered, so `set_objective` is a 404 until then: in flow 1, register the first structure first ([build.md](build.md#structure)). Without one, `loss_report` falls back to balanced losses. Compare only reports with the same `objective.effective_hash`.

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
| `near_cuts`: wrong cases just below or above a cut whose expected outcome is the next band (`lower_it`, `raise_it`) | Sweep that cut (`sweep_cut`, below) and move it to the best point | No (numbers) |
| `by_mistake`: the loss sits in one cell (`expected` → `got`) between two outcomes, with no single suspect question | Re-weight the inputs of the aggregation that separates those outcomes by hand (limits below), or reorder its rules | No (numbers) |
| `by_question`: one question carries a large share of the loss | Rewrite that question's instructions or criteria from the cases' reasons; keep its name if its meaning stays. Read its `flip_effects` first | Yes (the rewritten question) |
| `missing_criterion`: wrong cases with no suspect, clustered on one theme | Add a question for that criterion and an input for it in the aggregation | Yes (the new question) |
| `answer_conflicts` on a case: a reviewer's `expected_answer` disagrees with the answer | Rewrite that question, as for `by_question` | Yes |
| `failed`: permanent failures | Check `reads` against the inputs the app always sends, and read an optional field through its parent | No |
| A label that contradicts the brief | Fix the label or the brief, not the structure: go back to `labels` (flow 1) or `brief_check` (flow 2) | No |

**Numbers come before questions.** Cut points and weights cost nothing to change. A question change costs engine calls on every case. So try the numbers the report points at before rewriting a question, and change one thing per candidate so the report can tell you what it did.

### Tune a cut with `sweep_cut`

`sweep_cut` re-decides a run's labeled items from the answers the run already holds, at every point one cut could move to, and judges each point by the objective as `loss_report` does. It makes no engine calls and spends nothing. Over MCP it takes its request under `body`.

- **Call it** with the run you just judged and the aggregation `near_cuts` names: `{run_ids, aggregation, at}`. `at` is the cut's current value. Leave `at` out to sweep each cut of the aggregation in turn, the others held where they are.
- **Read each cut's `best`** (the response has one entry per swept cut in `cuts`). It is the lowest loss per case, with `change` (negative is better), and how many cases it `fixed` and `broken` against the current cut, with ids in `fixed_examples` and `broken_examples`. The cut moves only between its neighbours (`bounds`). `points` lists the loss across the range, so you can see whether the best is a broad valley or a single spike.
- **Check `cross_validation` before trusting it.** `best` is in-sample: chosen on the same cases it is scored on. `cross_validation` chooses the cut again without each fold of item groups and scores it on that fold. When `holds` is false, the in-sample gain is likely noise: leave the cut where it is. It is null when fewer than 2 item groups on either side of the cut can move (cases the band below gets right, and cases the band above gets right): too little evidence to check what moving the cut costs. Treat null like `holds: false`. Stored cases are grouped by content; a dry run's items only by identical answers, so near-duplicates in a dataset can still make it optimistic.
- **A 422 `REQUEST_TOO_LARGE`** means the runs are too many for the structure's size: sweep fewer runs at a time, or name one cut with `at`.
- **Score it like any candidate.** Put `best.at` in the structure and score it, then keep it only when `loss_report` accepts it. In flow 2 a numbers-only candidate over `cases` makes no engine calls. In flow 1 the re-score is a dry run that asks every question again, so batch the cuts that hold into one candidate.

### Set weights by hand, within limits

Automatic weight fitting isn't offered yet. You may move weights by hand, within these limits:

- **Small steps.** Move a weight by at most half its current value per candidate, and never to 0.
- **A reason for each.** Name the brief's rule or the report's evidence (a mistake cell, a flip effect) behind each change, in the commit message.
- **Signs and `invert` stay.** Never flip a weight's sign or an input's `invert`: that changes what the question means to the decision. Rewrite the question instead.
- **One aggregation per candidate,** so the report can tell what the change did.
- **The usual keep rule.** Keep it only when `loss_report` accepts it and it adds no severe mistake on any label.

Weights tuned by eye on a small set fit noise faster than a cut does, since several move together and nothing cross-validates them. On a small set, prefer moving a cut or rewriting a question.

### Read a flip's cost before rewriting a question

Each listed mistake in the report names its `fixing_flips`, the questions whose opposite answer alone would fix it. On the worst 25 mistakes, each flip also has a `flip_effects` entry, in the same order, unless the report ran out of its budget for working them out (a flip without one has no entry, so match entries by `question`). The entry assumes the question is reworded so that every case answering it the way this case does now answers the other way. It counts the wrong cases that would fix (`fixes`, this one included) and the right cases it would break (`breaks`, a few named in `broken`), with the net `loss_change`. A flip that breaks more than it fixes points at a narrower rewording, one that only moves answers like this case's, or at a missing criterion rather than a bad question.

How much a re-score costs depends on the flow:

- **Flow 1.** A dry run over `items` asks every question again, so batch the number changes the report supports into one candidate.
- **Flow 2.** A run over `cases` reuses the stored answers for questions the candidate didn't change, so a numbers-only candidate makes no engine calls (`cost.engine_calls` stays 0).

**Cost.** Every run's response carries `cost` (money; `usd_micros` is millionths of a dollar). Agree a budget with the person before the loop. `get_usage` reports the decision's spend over a window, including dry runs.

## When to stop

Stop when the person says the decision is good enough, or when `target_met` is true. When three candidates in a row are dropped, stop and report to the person what the report shows. The brief or the labels are usually the problem, not the structure.
