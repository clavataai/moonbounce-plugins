# Flow 2: improve a decision in use

The structure exists and real feedback is the main signal. There is no "shipped version" pointer: each evaluation records whichever structure hash the caller sent.

| Stage | Does | Leaves | Done when |
|---|---|---|---|
| `brief_check` | Checks the brief against the current structure and recent feedback: each clause covered, each question traced to a clause, no reviewer reason or `expected_outcome` contradicting it. Drafts one from the structure when none exists | Brief edits, its new structure hash and date | **Stop:** the user confirms the gaps or the drafted brief |
| `baseline` | Picks the structure: the latest with primary traffic, or one the user names; scores it | Baseline loss report | A baseline exists |
| `cases` | Lists the labeled cases, whatever structure they ran under; adds targeted synthesis (rare outcomes, unseen edge cases, new policy clauses, regression probes) | Case list, synthetic JSONL | The usable cases are known, with drop reasons |
| `refine` | The refine loop, backtesting each candidate against the last kept version on the labeled cases | Accepted versions, loss reports | **Stop:** good enough, or the target is met |
| `ship` | Registers the chosen structure; the caller's code starts sending its hash | Decision file, registered hash | **Stop:** the user decides to ship. Blind spot checks measure it afterwards |

At a **Stop**, show the person what the stage left and wait for their answer before moving on. When a stage is done, set the brief's `stage` to the next one and its `date` to today, then commit.

The brief is context here. When it exists, it settles which side is right when feedback conflicts, and a policy change updates it first. Reports show human-labeled and synthetic cases apart, and real data decides when they disagree.

Use the key for the environment whose traffic you are improving, usually production. Every stage reads, except `ship` and dry runs.

## Stages

### `brief_check`

Read `moonbounce/<decision_id>.brief.md` and the decision file. When there is no brief, draft one from the structure, in the sections [build.md](build.md#files) lists:

- each question's instructions and criteria become clauses;
- the result's outcomes become the Outcomes section.

Ask the person about every clause you had to infer.

Set or update the front matter: `flow: improve`, `stage: brief_check`.

Then check the brief against what the decision does and what people said:

- **Every clause is covered:** some question or rule applies it.
- **Every question traces to a clause:** a question with no clause is a gap in the brief or a question to drop.
- **Nothing contradicts it:** read recent `reasons` and `expected` from `list_cases` (newest first), and the feedback export (REST `GET /v1/decisions/{decision_id}/feedback`, not an MCP tool) when you have a key. Look for a reason or an `expected_outcome` that contradicts a clause.

Show the person the gaps and contradictions. A contradiction is settled in the brief first: the person decides whether the clause or the feedback is wrong. Record the structure hash you checked against and the date in the front matter.

### `baseline`

The baseline is the structure with the latest primary traffic, unless the person names another. To find it, call `list_evaluations` with `assignment_mode: primary` and `limit: 1`, and read `version.structure_hash`. With no primary traffic yet (just after flow 1, say), use the newest imported example's (`assignment_mode: example`), or the repo's decision file once the person confirms it. Read the structure back with `get_structure`, and check that the repo's decision file is the same structure (`mb structure hash --local moonbounce/<decision_id>.json`).

If the baseline isn't registered (the repo's decision file, say), register it first (`register_structure`). Score it on the cases with `evaluate` with `cases` and `loss_report`, with one run per label source: `cases.label_source` takes one value, so `human`, `unknown` and `synthetic` are three runs. Check `list_cases` with each `label_source` first and skip a source with no cases, since a case run that matches nothing is a 400 `NO_CASES`. Keep the reports and run ids. The baseline is the first kept version, so it is what the first candidate must beat. If the objective isn't stored in this environment, or differs from the decision file's, store it first (`set_objective`).

### `cases`

List the labeled cases with `list_cases`, whatever structure they ran under. Set `outcomes` to the outcomes either version can produce.

Read `excluded`: it says which evaluations with feedback aren't cases, and why. Usable cases have both of these:

- stored content;
- an outcome label that either version's result can produce. A candidate that drops an outcome is wrong on its cases, not excused from them.

Disagreements drive changes and agreements guard against regressions, so both stay in the backtest.

Then add targeted synthesis where real cases are thin:

- rare outcomes;
- edge cases the brief names that no case covers;
- new policy clauses;
- regression probes for a change you plan.

Use a fresh case writer and a fresh labeler, as in flow 1's [`cases`](build.md#cases) and [`labels`](build.md#labels) stages. Write the result to `moonbounce/<decision_id>.synthetic.jsonl`, with `label_source: "synthetic"` (or `human` where the person settled the label). These lines stay in the repo. They are scored with dry runs over `items`, apart from the real cases, and never imported among the real cases.

Done when you can say how many usable cases there are, how many were dropped and why, and what the synthetic set adds.

### `refine`

Run [the refine loop](refine.md). Backtest each candidate against the last kept version (at first, the baseline) on the same cases, with the runs a preview would make but kept so you can page them:

1. **The kept version**: `evaluate` with `cases`, its registered hash in `moonbounce.structure_hash`, and `refresh` set to its questions that the candidate rewords or drops. Reuse its runs from the last comparison only when the cases, the `refresh` set and the kept version are all the same and the runs haven't expired. A different `refresh` set changes which answers the base replays, so score the base again.
2. **The candidate**: `evaluate` with `cases`, the structure inline, over the same cases.
3. **Judge** with `loss_report`, the kept version's runs as `base`.

Run each step once per label source that has cases, as in `baseline`, and judge two reports:

- **The deciding report**: the `human` runs. When there are fewer than about 20 human-labeled cases, too few to tell, add the `unknown` runs to both sides of the same `loss_report` (it takes several `run_ids` a side) and tell the person which labels decided.
- **With no human-labeled or `unknown` cases at all** (only synthetic labels, just after flow 1 for instance), the `synthetic` runs decide. Tell the person the decision has no real labels yet, and that review of real traffic is what will test it.
- **Reported apart**: the `synthetic` runs, and the `unknown` ones, whenever they didn't decide.

Score the synthetic set the same way, with dry runs over `items`, judged by its own `loss_report`. When the deciding report and the rest disagree, the deciding report wins.

Register each version you keep (`register_structure`), so it can be the next base. `preview` gives the same comparison in one call when you only need the headline: it has no run ids to page and doesn't split by label source.

**Attack rounds.** The stored cases cover only what traffic has sent so far. A reworded question or a moved cut can open a weakness that no case touches. So after each kept version, a red team attacks what that version changed, as in flow 1's [`refine`](build.md#refine). The baseline changed nothing: the targeted synthesis at `cases` covers it, and the first round comes after the first kept candidate.

1. **Write.** Brief a fresh red team with the brief, the kept structure, and the diff from the version before it: "Write 10 to 30 cases this change is likely to get wrong, whose right outcome the brief settles: paraphrases near a moved cut, cases that hinge on a reworded question's criteria, near-misses for a new question, and cases the old version got right that the change could break. Use only the inputs the brief lists." It writes cases and intended labels apart, with `metadata.origin: "red"`.

   Any model can be the red team. When the person has another model or vendor available, recommend using it, and record which model wrote each round.
2. **Label.** The labeler labels the round blind. Take disagreements and unsure cases to the person.
3. **Score.** Score the kept version on the round with a dry run over `items`, and report it by attack. The round was written to break this version, so it doesn't gate the version it attacked.
4. **Keep.** Add the round, with its settled labels, to `synthetic.jsonl`, so later candidates are scored on it.

Red cases are synthetic: they are reported apart and never outweigh real labels in deciding a keep. One rule still applies to them: a candidate that makes a new severe mistake on any label, red cases included, is dropped ([refine.md](refine.md#keep-or-drop-a-change)). So a round can block a dangerous change but can't carry one through.

Also stop and ask the person when a round finds nothing wrong, or after three rounds.

### `ship`

The person decides to ship. Then do these in order:

1. Register the chosen structure in each environment it will run in (`register_structure`), and check the returned hash against the one you scored.
2. Check the binding (`get_binding`), and store the decision file's objective (`set_objective`; `unknown_outcomes` must be empty).
3. Put the decision file in the caller's code, on a branch, for the person to merge. Evaluations start recording the new hash once the merged code sends it.

Set the brief's `structure_hash` to the shipped hash and `stage` to `done`.

Blind spot checks, which reviewers answer on production traffic after the version ships, are the measure of what was shipped. The next run of flow 2 reads them as cases.

## Going back

- **A policy change or a contradiction found at `refine`:** go back to `brief_check`. Then re-run `baseline`, because the objective or the labels may have moved.
- **New feedback arriving mid-loop:** re-run `cases` before the next candidate.
