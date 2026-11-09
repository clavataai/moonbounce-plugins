# The refine loop

Both flows end in this loop. The build flow ([build.md](build.md)) runs it over the repo's dataset, and the improve flow ([improve.md](improve.md)) runs it over the decision's labeled cases. Each pass scores a candidate under the decision's objective, reads the loss report to see where the loss sits, edits the structure and scores again. A change is kept only when the loss report accepts it. The loop stops when the person says the decision is good enough.

## Roles

The **author** edits the structure, and you (the orchestrator) score it. Scoring is mechanical, so you can do it yourself, or give it to a subagent on a smaller, cheaper model ([The scorer](#the-scorer) below). Brief a fresh author each round with the brief, its notes and the round's report summary: a long-lived author re-reads every earlier round and re-emits the whole structure each time. The author never scores its own version and never reads a set it hasn't been given:

- In the build flow, the author is given the repo's dataset and never the blind set.
- In the improve flow, the author is given the decision's cases.

Give the author the summary's lines and `report_url`, and what the round's change needs from the full report: the roll-ups it aims at and the worst cases they list (`detail: "full"`), or the rows behind a cell (`cases`), each with its `reason` from the dataset or `list_cases`. Reasons are often the best clue for a rewrite. Pass on only the parts the change needs, never the whole report.

Put these in the author's prompt:

- **Validate by uploading.** Call `create_upload` with `{kind: "structure"}`, then `curl -sS -X PUT --data-binary @<candidate file> <url>`: it answers the candidate's `structure_hash`, or 400 `CONTENT_INVALID` with the issues to fix ([Upload a file](tasks.md#upload-a-file)). Hand back the file and its hash.
- **Describe, don't quote.** State a question's criteria as the kind of request it covers, not the wording of the cases that broke it: a list of a red-team round's phrasings fits that round and can miss the next one. Examples in criteria are fine as illustrations of the kind, not as the set.
- **Two facts compared aren't a judgment.** Whether two addresses are the same, or one letter apart, is a comparison the model makes unsteadily, and it flips between runs; a table of every value pair (one rule per country) is the same fact written out longhand. Make both values attributes and compare them in the structure: a feature comparison, with `normalize: "email"` for addresses or `"case"` for codes ([Facts are attributes](design.md#facts-are-attributes-judgment-is-a-question)). When one value is only in the message text, ask the person before relying on a raw field the app doesn't send yet (`state.requester.typed_email`, derived with `from`); never ask for a derived boolean, which hides the comparison in the app's code, outside the hash.

### Candidates

Each candidate is one change (except the batched cuts below), kept in `moonbounce/scratch/<decision_id>/` until it is kept. The repo's decision file holds only the last kept version: copy a candidate there once it is kept, never before it is scored.

- **The author edits a copy, not a rewrite.** It copies the version it starts from to `moonbounce/scratch/<decision_id>/<round>-<letter>.json` (`cp`) and edits that copy in place, so it writes only the text that changes, never the whole structure.
- **Each candidate goes up once.** The author's upload answers its `structure_hash`, and every run of the candidate names that hash while the upload lives (about a day; then upload the file again, and the hash is the same). A candidate whose hash is its base's is its base: don't score it.
- **Independent fixes are siblings.** When a round has several fixes that don't depend on each other (they change different features: different questions, attributes or aggregations by name; two fixes inside one aggregation are one candidate or a stack), each candidate starts from the last kept version and makes its one change, and each is scored against that version. A dropped one then affects only itself, and siblings can be scored in parallel.
- **A sibling that passes is accepted, not yet kept.** Siblings are each judged against the same base, so keeping them one after another could replace a better version with a worse one. Register nothing and leave the decision file alone until the round is settled:
  - one sibling accepted: it is the round's kept version;
  - several accepted: have the author combine them, as a copy of the last kept version with each accepted change made in it, and score the combination against the best single one (lowest loss per case), to catch the ways two changes interact. Keep the combination when the usual rule accepts it against that one; otherwise keep the best single one.

  In the improve flow, score the best single sibling again as that base, with `refresh` set to the other siblings' changed questions, as for any change to `refresh` ([improve.md](improve.md#refine)); its runs as a candidate replayed those questions' stored answers.
- **Stack candidates only when a change needs the one before it.** In a stack, each is scored against the one before. When one is dropped, the ones built on it carry its change, so drop them too or have the author rewrite them from the last kept version.
- **When a candidate is kept,** register it by its hash ([Keep or drop a change](#keep-or-drop-a-change)) and `cp` its file over the decision file.

### The scorer

A scoring subagent runs the runs and the report, and hands back numbers, not reports. Put these in its prompt:

- Call `create_run` with `results: "failed"`: the response lists only the failed items, beside the `run_id`, `counts` and `cost`. The run keeps every result, and `loss_report` reads them by `run_id`.
- Call `loss_report` with `detail: "summary"`: `accepted`, `reasons`, `new_severe_human`, `changes` as counts, `objective`, each side's `n`, `loss_per_case`, `target_met`, `macro_f1`, `accuracy`, its three costliest mistake cells and how many there are, `excluded` and `report_url`. Over MCP it is the default; over REST or the SDK the default is `full`, so send it. Never send `show`: the chart is drawn only for the report the orchestrator shows the person.
- For the [round report](../SKILL.md#what-the-person-sees), call `loss_report` once more over the same runs with `detail: "full"` and build its rows from it: the priority outcome's precision and recall (`outcomes`), and the loss matrix (each cell's count from `confusion`, `(was n)` from the base's, and its share of the loss from `by_mistake`). Hand back those rows as the round report's table, not the report.
- Hand back the run ids with their structure hash and expiry, each failed item's id and error code, the summary's numbers as a few lines, the round report's table and `report_url`. Never restate a report, copy its JSON or write it to a file: the full report stays one call away (`detail: "full"`) and in the chart.
- Look at particular mistakes only when the orchestrator asks for them: rows through `cases` with a filter (`cell`, `severe`, `changed`, ...), or `detail: "full"` for the roll-ups and the worst mistakes with their `fixing_flips`. Hand back only what was asked for.
- **How a question answered particular cases.** The full report's worst mistakes carry the answers of their fixing flips and conflicting questions (`answers`): read those first. When the author needs every question's answer on a few chosen cases, run just those items again (in the build flow, their lines picked by id into a scratch file, as [Looking at the data](#looking-at-the-data) shows, and uploaded; in the improve flow, `cases: {evaluation_ids}`) with `include: ["answers"]` and `results: "all"`, and hand back the answers as a small table. Never run the whole dataset again to collect answers: a re-run is another draw of a noisy model, and it costs a full scoring.
- Write nothing to files but what the orchestrator asks for, and never copy a tool's result into a file by retyping it.

## Score a version

Never score by hand, and never pass the results back yourself. Pass `run_id`s so the results stay out of your context. Over MCP, `create_run` and `loss_report` take their request under `body`. [Score a dataset](tasks.md#score-a-dataset) has the full request shapes.

- **Every run you only score** takes `results: "failed"`, so its response lists only the failed items (to resend or look into) and not a result per item. `counts`, `cost` and the `run_id` cover every item.
- **A dataset (the build flow, and the improve flow's synthetic cases).**
  - Upload the file once and call `create_run` with `items_from: {upload_id, offset}`, one run per 200 lines (offsets 0, 200, 400…), the same slices for every version ([Score a dataset](tasks.md#score-a-dataset)). A run of `items_from` is always a dry run, so it adds no evaluation of an unshipped version to the decision.
  - Name the version by `structure_hash` at the top level of `body`: the last kept version's, or a candidate's from its upload, which a dry run takes while the upload lives. `decision_id` is the path parameter; a run takes no `moonbounce` object.
  - Upload the file again when it changes (a red-team round, a settled label) or its upload has expired (404 `UPLOAD_UNKNOWN`). A version that needs content the lines lack (a caller-supplied attribute without a default, a state field) is refused naming the item: give a new attribute a `default` or fill it from the state (`from`) and it needs none.
- **Stored cases (the improve flow).**
  - Call `create_run` with `cases` in place of `items`: either `{evaluation_ids}` from `list_cases`, or its filters with `limit` and `cursor`. Cases are always dry runs. A run takes at most 200 cases, so page larger sets and score both versions over the same pages.
  - Each case reuses the answers it stored for questions the version didn't change.
  - Filter by `label_source` to score human-labeled cases apart from the rest: real labels decide in the improve flow.
  - A preview (`create_preview`) is two of these runs and a `loss_report` in one call, over the newest 500 cases. It returns compact reports with no `run_id`s to page and no split by label source, so the improve flow scores with the runs themselves.
- **Judge.**
  - Call `loss_report` with `{candidate: {run_ids}, base: {run_ids}}`. Over MCP it answers the summary (over REST or the SDK, send `detail: "summary"` for it): `accepted` and `reasons`, each side's `loss_per_case`, `macro_f1`, `accuracy`, `n` and its three costliest mistake cells, `changes` as counts, `objective`, `excluded` and `report_url`. Read the verdict from it. `detail: "full"` answers the whole report, with the roll-ups below; ask for it only when you need them, and pass on the parts you need, not the report.
  - With a base, `changes` says how many cases the candidate `fixed`, `broken` (right under the base, wrong now) and `moved` (to another wrong outcome); the full report also lists each move by cell in `moves` (the commonest 1,000; `moves_total` counts them all). A lower loss can hide what it broke, so read `broken` before keeping a change, and list them with `cases: {filter: {kind: "changed"}}`, which returns every row whose outcome changed: the broken ones are those whose `base.got` equals `expected`.
  - When both sides are runs it also returns `report_url`, the report as a chart in the web app: print it in the round report (SKILL.md), never fetch it.
  - Page through the cases behind a roll-up with `cases: {filter: {kind: "wrong"}}` (or `cases: {filter, sample: n}` for a spread of n of them). The other kinds are `severe`, `changed`, `missing_criterion`, `{kind: "cell", expected, got?}`, `{kind: "question", question}`, `{kind: "near_cut", at}` and `{kind: "like", item_id}` (the rows nearest one row).
  - Price base and candidate on the same items. Items the base lacks are counted in `excluded.no_base` and left out, so when the dataset grows, score the base on the new items too and add that run to its `run_ids`; when a line already scored changes (its content or label), score the base again over the new upload.
  - Runs expire after about a day. Keep the run ids with their structure hash, the upload and offset each read and their expiry (the build flow keeps them in `work/runs.json`), and re-score the last kept version when its runs have expired.
- **Look at the data.** Choose how much to read, and how, from [Looking at the data](#looking-at-the-data) below.
- **Report.** In a host that draws MCP Apps, call `loss_report` with `show: true` for the report you show, from the main conversation, not a subagent, so the chart draws, once per report; every other `loss_report` call leaves `show` out and draws nothing; in a terminal the scorer's table is the report ([Show one report](../SKILL.md#what-the-person-sees)). Show the person each round in the [round report](../SKILL.md#what-the-person-sees) format, base beside candidate: loss per case and whether it was accepted (with `reasons` when not), `macro_f1`, the precision and recall of the outcome the objective's `priority` names (from `outcomes`), and the loss matrix (`confusion` and `by_mistake`), which shows where the loss sits. The summary has the headline numbers; the matrix and the priority outcome's line come from one `detail: "full"` call over the same runs, read for those numbers only (a scorer hands back the finished table: [The scorer](#the-scorer)). Loss decides what is kept, because it is the objective the person set; the other numbers help them see what changed.
- **Failures.**
  - An item whose model calls fail permanently (unreadable content, a model refusal) is counted as its dearest mistake.
  - A retryable failure (`NOT_STARTED`, `ENGINE_UNAVAILABLE`) is left out as `excluded.failed_transient`. Resend those items in a new run with a new Idempotency-Key, and list both runs.
- **Same objective.** Store the objective (`set_objective`) before the first scoring. A new decision exists only once a structure is registered, so `set_objective` is a 404 until then: in the build flow, register the first structure first ([build.md](build.md#structure)). Without one, `loss_report` falls back to balanced losses. Compare only reports with the same `objective.effective_hash`.

## Looking at the data

Read as much as you need to understand what the version gets wrong and why, and choose the way that fits. What you read stays in your context, and in the author's when you pass it on, so weigh what a read tells you against what it costs: a few dozen items are cheap, a few thousand crowd out everything else. These ways go from the least data to the most, and they combine:

| Way | What it gives you | Good for |
| --- | --- | --- |
| **The summary** (no items) | `loss_report` with `detail: "summary"` (over MCP, the default): the verdict, the headline numbers and each side's three costliest mistake cells | Whether to keep the candidate, and which cell to look at first |
| **The roll-ups** (no items) | `loss_report` with `detail: "full"`: `confusion` and `by_mistake` (where the cases and the loss are), `by_question`, `near_cuts`, `missing_criterion`, `outcomes`. Rows of `by_mistake`, `by_question`, `near_cuts` and `missing_criterion` list a few example ids; for a `confusion` cell, page it with `{kind: "cell"}` | Seeing where to look before reading anything. Often enough to choose a numbers-only change |
| **The rows behind a roll-up** | `loss_report` `cases: {filter}`: `wrong`, `{kind: "cell", expected, got}`, `{kind: "question", question}`, `{kind: "near_cut", at}`, `missing_criterion`, `severe`, `changed` (base and candidate differ), `right` or `all`, paged with `limit` and `cursor`. Each row is an id, its label, what each version gave and its loss | Every case in one mistake cell, or every case one question is blamed for |
| **A spread sample** | `cases: {filter, sample: n}`: n of the filter's rows, split across expected outcomes (at least one each when n allows) and spread over how the candidate answered them, the same rows for the same report and filter | A representative look at a large filter, and a check of what the version gets right (`{kind: "right"}`, `sample: 5` to 10) so a change isn't fixing the wrong thing |
| **The cases like one** | `cases: {filter: {kind: "like", item_id}, limit}`: the right rows nearest that row by how the candidate saw them (its answers, attributes and cut value), nearest first, each with its `distance`, paged (not sampled). `distance` is relative: the nearest rows may still not be close. `among: "wrong"` ranks the wrong rows instead, mistakes the version answered alike (not necessarily for one reason); `metric: "pattern"` (three or more questions) compares which way each question leans, whatever its confidence, as `similarity` (1 when every question leans alike) | Before changing anything for one wrong case: the right cases the change is most likely to break |
| **The items themselves, by id** | In the build flow, rows carry `item_id`s: pick those lines from the dataset and the red rounds with jq (below). In the improve flow, rows over case runs carry evaluation ids: `list_cases` with `evaluation_ids` (comma-separated) and `include=content`, 50 a call | Reading the content, labels and reasons behind the rows you chose |
| **Everything** | `{kind: "all"}` paged, the whole dataset file, or every case with `list_cases` paged | A small set (a few hundred items), or when nothing narrower answers your question. For a large one, a script that reads the file or the REST API keeps the bulk out of your context, and you read what it finds |

Picking dataset lines by id (they come back in file order): write the ids as a JSON array to `moonbounce/scratch/<decision_id>/pick-ids.json` with the file tool, fresh each time, never into the command ([Stay inside the repo](../SKILL.md#where-a-flow-stands)). The second command is for while a red round's cases aren't in the dataset yet: it reads that round's file after the dataset.

```sh
jq -cs '.[0] as $ids | if ($ids | type) != "array" then error("the ids file is missing or not a JSON array") else .[1:][] | select(.item_id | IN($ids[])) end' moonbounce/scratch/<decision_id>/pick-ids.json moonbounce/<decision_id>.examples.jsonl
jq -cs '.[0] as $ids | if ($ids | type) != "array" then error("the ids file is missing or not a JSON array") else .[1:][] | select(.item_id | IN($ids[])) end' moonbounce/scratch/<decision_id>/pick-ids.json moonbounce/<decision_id>.examples.jsonl moonbounce/work/<decision_id>/red-<n>.jsonl
```

A usual round: the summary, the roll-ups, then the rows of the costliest cell or the most-blamed question, then those items by id, plus a spread sample of right rows. Before every change aimed at particular wrong cases, including the fixes after a red-team round, read the right cases nearest each one (`like`, 5 to 10 rows) and their content, and check the change still gives them the right outcome. Give the author what you read that bears on the change, with labels and reasons, and the roll-ups. Read more whenever you're not sure why the version gets something wrong.

## Keep or drop a change

Keep a candidate when both of these hold:

- **The report accepts it.** `accepted` is true, with the last kept version as the base: lower loss per case, no new severe mistake on a human-labeled case, and at least 80% of the items scored (resend the retryable failures when fewer were).
- **It adds no severe mistake on any label.** The no-new-severe-mistake rule counts only `human` labels, so when most labels are synthetic, also page the severe cases (`cases: {filter: {kind: "severe"}}`). Drop the candidate when any row has `base.severe: false`: a case the base got right is now one of the objective's severe mistakes, whatever its label.

**A repair after a red-team round (the improve flow).** Real labels may already be right, so a candidate that only fixes red cases leaves their loss unchanged, and `loss_report` won't accept it. Keep it when the synthetic report (the red rounds included) accepts it and, on the `human` runs and on the `unknown` runs, each judged apart, loss per case is no higher and no case the base got right is now a severe mistake. Its verdict is `kept (red-team repair)`; it applies only on a return from `red_team`.

The first structure that validates and scores is the first kept version: it is the base the next candidate must beat. `loss_report` never accepts a candidate without a base, so don't wait for the target to keep a first version. A target, when the person set one, is a reference point: it is neither when to keep nor when to stop.

Register each kept version as soon as it is kept, before the next round or stage: `register_decision` with `{decision_id, structure_hash}`, the hash its upload answered, while the upload or one of its runs lives, so the structure isn't sent again. Its hash then works as a base and at import, and stopping at any point leaves the server with it. Its `version.number` is the version's name from then on (vN in every report). Save it in the decision file. When the brief's `commits` names a branch, commit it there with its version number, structure hash and loss per case in the message. The next candidate starts from the last kept version, or from a dropped one when the author judges that one closer.

Models answer the same case differently from run to run. When a verdict hinges on one or two cases, score both versions twice. Small sets are only indicative.

## From evidence to a change

Read the report top down. Each roll-up points at one kind of change. Before re-weighting, regrouping or reordering rules, reread [Design a structure](design.md): the right aggregation often matters more than its numbers.

| The report shows | Change | Spends model calls |
| --- | --- | --- |
| `near_cuts`: wrong cases just below or above a cut whose expected outcome is the next band (`lower_it`, `raise_it`) | Sweep that cut (`sweep_cut`, below) and move it to the best point | No (numbers) |
| `by_mistake`: the loss sits in one cell (`expected` → `got`) between two outcomes, with no single suspect question | Re-weight the inputs of the aggregation that separates those outcomes by hand (limits below), or reorder its rules | No (numbers) |
| `by_question`: one question carries a large share of the loss | Rewrite that question's instructions or criteria from the cases' reasons; keep its name if its meaning stays. Read its `flip_effects` first | Yes (the rewritten question) |
| `missing_criterion`: wrong cases with no suspect, clustered on one theme | Add a question for that criterion and an input for it in the aggregation | Yes (the new question) |
| `answer_conflicts` on a case: a reviewer's `expected_answer` disagrees with the answer | Rewrite that question, as for `by_question` | Yes |
| A choice or score in `fixing_flips` whose `flip_to` names its own label (the same answer, made sure) | Its label was right; what read it went wrong. If a rule on its `confidence` (or a cut on a score's number or an option's probability) caught it, move that threshold (with `sweep_cut` when it is a `bands` cut), or route on the label (`match`, `is`) instead; else sharpen its criteria | No (a threshold), or yes (criteria) |
| Mistakes between outcomes that several yes/no questions route, which say yes together or all say no on the wrong cases (read their `answers`) | Ask one `choice` instead: the categories exclude each other ([Choose the kind of question](design.md#choose-the-kind-of-question)) | Yes (a new question) |
| A `weighted_mean` of graded yes/no questions ("mild?", "serious?") whose `near_cuts` mistakes a sweep doesn't clear | Ask one `score` on a rubric and cut it with `bands` | Yes (a new question) |
| `by_question` or `answer_conflicts` on a question about a fact the state already holds (an amount, days, a count, a flag) | Make the fact an attribute derived from its state path (`from`) that a rule or cut compares exactly, and drop the question ([Facts are attributes](design.md#facts-are-attributes-judgment-is-a-question)). The app's call doesn't change, and stored cases recompute it from their saved state, so score it like any candidate. If the app already sends that attribute in `attribute_values`, see [Switching an attribute the app already sends](design.md#facts-are-attributes-judgment-is-a-question) first | Fewer: the question goes |
| `failed`: permanent failures | Check `reads` against the inputs the app always sends, and read an optional field through its parent | No |
| A label that contradicts the brief | Fix the label or the brief, not the structure: go back to `labels` (build) or `brief_check` (improve) | No |
| A mistake the brief doesn't settle (the version's answer and the label differ, and no clause decides which is right) | Ask the person before changing the structure: in the build flow, put a few probes about it to them ([build.md](build.md#brief)), labeled blind; the answers change the brief | No |
| A case wrong in two or more kept versions in a row, especially one labeled `unknown` | Take it to the person as a possible wrong label before another change aims at it: show the case and its reason, without your view first | No |
| One question rewritten in two or more rounds and still in `by_question` | It likely carries too many criteria: split it into narrower questions, each one call the model can make steadily, before rewriting it again | Yes (the new questions) |

**Numbers come before questions.** Cut points and weights cost nothing to change. A question change costs model calls on every case. So try the numbers the report points at before rewriting a question, and change one thing per candidate so the report can tell you what it did. The one exception: in the build flow, where every re-score asks every question again, batch the cut moves that each passed `sweep_cut`'s `cross_validation` into one candidate.

**A new attribute in the improve flow.** An attribute derived from the state (`from`) is recomputed from each stored case's saved state, so a case run scores it directly. A case whose state lacks the path takes the attribute's `default`; without one it fails as `CONTENT_INVALID`, so give a derived attribute a default when older traffic may lack the field. Turning a derived attribute back into one the app sends works on stored cases too: each case's derived value stands in for the supplied one, though `list_cases` content leaves it out (it shows only what the caller sent). An attribute the app would send in `attribute_values` (a fact not in the state) is different: stored cases don't carry it and fail on every case. Score that candidate over uploaded copies instead: page the cases with `list_cases` `include=content` (50 a page) and write two files with jq, with the same `item_id`s and each case's label and `label_source`. The kept version's copy, `work/cases-base.jsonl`, has `{item_id, state: .content.state, claims: [{kind: "expected_outcome", value: .expected}], label_source}` plus `attribute_values` as stored (a version refuses an attribute it doesn't declare). Add `attribute_values` only when the case has them: `list_cases` leaves it out of `content` for a case with none, and `evaluate` refuses `attribute_values: null` with a 400. In jq, `+ (if .content.attribute_values then {attribute_values: .content.attribute_values} else {} end)`. The candidate's, `work/cases-with-<attribute>.jsonl`, adds the new attribute to `attribute_values`. Upload each copy, score each version over its own, and report it as a scoring of copies, not of the stored cases.

### Tune a cut with `sweep_cut`

`sweep_cut` re-decides a run's labeled items from the answers the run already holds, at every point one cut could move to, and judges each point by the objective as `loss_report` does. It makes no model calls and spends nothing. Over MCP it takes its request under `body`.

- **Call it** with the run you just judged and the aggregation `near_cuts` names: `{run_ids, aggregation, at}`. `at` is the cut's current value. Leave `at` out to sweep each cut of the aggregation in turn, the others held where they are.
- **Read each cut's `best`** (the response has one entry per swept cut in `cuts`). It is the lowest loss per case, with `change` (negative is better), and how many cases it `fixed` and `broken` against the current cut, with ids in `fixed_examples` and `broken_examples`. The cut moves only between its neighbours (`bounds`). `points` lists the loss across the range, so you can see whether the best is a broad valley or a single spike.
- **Check `cross_validation` before trusting it.** `best` is in-sample: chosen on the same cases it is scored on. `cross_validation` chooses the cut again without each fold of item groups and scores it on that fold. When `holds` is false, the in-sample gain is likely noise: leave the cut where it is. It is null when fewer than 2 item groups on either side of the cut can move (cases the band below gets right, and cases the band above gets right): too little evidence to check what moving the cut costs. Treat null like `holds: false`. Stored cases are grouped by content; a dry run's items only by identical answers, so near-duplicates in a dataset can still make it optimistic.
- **A 422 `REQUEST_TOO_LARGE`** means the runs are too many for the structure's size: sweep fewer runs at a time, or name one cut with `at`.
- **Score it like any candidate.** Put `best.at` in the structure and score it, then keep it only when `loss_report` accepts it. In the improve flow a numbers-only candidate over `cases` makes no model calls. In the build flow the re-score is a dry run that asks every question again, so batch the cuts that hold into one candidate.

### Set weights by hand, within limits

Automatic weight fitting isn't offered yet. You may move weights by hand, within these limits:

- **Small steps.** Move a weight by at most half its current value per candidate, and never to 0.
- **A reason for each.** Name the brief's rule or the report's evidence (a mistake cell, a flip effect) behind each change, where the person will read it: the commit message when you commit, else your report to them.
- **Signs and `invert` stay.** Never flip a weight's sign or an input's `invert`: that changes what the question means to the decision. Rewrite the question instead.
- **One aggregation per candidate,** so the report can tell what the change did.
- **The usual keep rule.** Keep it only when `loss_report` accepts it and it adds no severe mistake on any label.

Weights tuned by eye on a small set fit noise faster than a cut does, since several move together and nothing cross-validates them. On a small set, prefer moving a cut or rewriting a question.

### Read a flip's cost before rewriting a question

Each listed mistake in the report names its `fixing_flips`, the questions whose flip alone would fix it: a noul's opposite answer, or a choice or score made certain on one of its options or levels, which `flip_to` names. Its current one is included: a low-confidence answer a rule escalated is fixed by the same answer made sure. `flip_to` lists them best first by their flip effects' `loss_change`, at most 10, with `flip_to_total` counting them all. A flip effect's `from` and `to` are then the noul's side, or the option or level. On the worst 25 mistakes, each flip also has a `flip_effects` entry, in the same order, unless the report ran out of its budget for working them out (a flip without one has no entry, so match entries by `question`, and for a choice or score by `question` and `to`). The entry assumes the question is reworded so that every case answering it the way this case does now answers the other way. It counts the wrong cases that would fix (`fixes`, this one included) and the right cases it would break (`breaks`, a few named in `broken`), with the net `loss_change`. A flip that breaks more than it fixes points at a narrower rewording, one that only moves answers like this case's, or at a missing criterion rather than a bad question.

`flip_effects` cover one question at a time. Before a rewrite, a new question or a cut move for a wrong case, also read its nearest right cases (`{kind: "like", item_id}`), then their content: a rewording has to keep giving them their answers. They are near in how the version answered them, not in their text, so read the content before trusting a match.

How much a re-score costs depends on the flow:

- **The build flow.** A dry run over the dataset asks every question again, so batch the number changes the report supports into one candidate.
- **The improve flow.** A run over `cases` reuses the stored answers for questions the candidate didn't change, so a numbers-only candidate makes no model calls (`cost.model_calls` stays 0).

**Budget.** Every run's response carries `cost` (money; `usd_micros` is millionths of a dollar). Before the loop, estimate and ask:

1. Dry-run five or so items (`limit: 5` in `items_from` or `cases`) and read the run's `cost.usd_micros`; divide by the items for a cost per item.
2. Multiply out the flow: in the build flow, items × (candidates you expect, about five to ten) plus the red-team rounds and the blind set. In the improve flow, a run over stored cases asks only the changed questions again (a numbers-only candidate costs nothing there), but the synthetic and red items are dry runs that ask every question, so add those items × candidates.
3. Tell the person the estimate in dollars, and ask for a cap.

Report each run's cost and the running total as you go (the **Cost** line of every [report](../SKILL.md#what-the-person-sees)), and stop and ask at the cap. `get_usage` reports the decision's spend over a window, including dry runs.

## When to stop

Stop when the person says the decision is good enough. When they set a target, report `target_met` each round, but it never stops the loop on its own. When three candidates in a row are dropped, stop and report to the person what the report shows. The brief or the labels are usually the problem, not the structure.
