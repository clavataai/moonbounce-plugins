# The improve flow: a decision in use

The structure exists and real feedback is the main signal. There is no "shipped version" pointer: each evaluation records whichever structure hash the caller sent.

| Stage | Does | Leaves | Done when |
|---|---|---|---|
| `brief_check` | Finds the structure in use, then checks the brief against it and recent feedback: each clause covered, each question traced to a clause, no reviewer reason or `expected_outcome` contradicting it. Drafts one from the structure when none exists | Brief edits, its new structure hash and date | **Stop:** the user confirms the gaps or the drafted brief |
| `baseline` | Picks the structure: the latest with primary traffic, or one the user names; scores it | Baseline loss report | A baseline exists |
| `cases` | Lists the labeled cases, whatever structure they ran under; adds targeted synthesis (rare outcomes, unseen edge cases, new policy clauses, regression probes) | Case list, synthetic JSONL | The usable cases are known, with drop reasons |
| `refine` | The refine loop, backtesting each candidate against the last kept version on the labeled cases | Accepted versions, each registered as it is kept; loss reports | **Stop:** the user says it's good enough |
| `red_team` | A red team (another model when the user has one) attacks what the settled version changed; the round is labeled blind, settled with the user, scored once and joins the synthetic set | `work/red-<n>.jsonl`, synthetic additions | **Stop** after each round: the user chooses `refine` (recommended when it broke a costly or severe mistake; the new version is attacked next), another round, or `ship` (recommended when it broke nothing worse than a normal mistake, or after three rounds) |
| `ship` | Registers the chosen structure; the caller's code starts sending its hash | Decision file, registered hash | **Stop:** the user decides to ship. Spot checks measure it afterwards |

At a **Stop**, show the person what the stage left and wait for their answer before moving on. When the question is how a case should go or which clause wins, ask what the person would decide before showing your recommendation, as the [Stop format](../SKILL.md#what-the-person-sees) says. When a stage is done, set the brief's `stage` to the next one and its `date` to today, and commit only when the brief's `commits` names a branch.

The brief is context here. When it exists, it settles which side is right when feedback conflicts, and a policy change updates it first. Reports show human-labeled and synthetic cases apart, and real data decides when they disagree.

Use the key for the environment whose traffic you are improving, usually production. Scoring is always dry runs, which store no evaluation; each run keeps its results and labels for about a day, then they are gone, as uploads are. Beyond those runs and uploads, only these steps write: registering a structure (`register_decision`, at `baseline` and for each kept version), storing the objective (`set_objective`, before the first scoring when the stored one differs from the decision file's, and at `ship`), a label the person has you correct at `brief_check`, and `ship`. Registering changes nothing the decision runs.

Two writes change what a decision in use does, so each is a **Stop**: say what it changes, and wait for the person's yes.

- **The objective** (`set_objective`) changes what every later report and suggestion scores by.
- **The model** (`set_model`) changes what the decision's live evaluations run on in that environment, from the next call, whatever structure the app sends. This flow never changes it on its own. Runs use the environment's model setting, and a run takes no model of its own, so a comparison needs a place with its own setting: the Test environment, which needs an `env:test` key (sign-in reaches Production only). Cases and uploads are per environment, so export the production cases to a file first (`list_cases` with `include=content`, written as **A new attribute in the improve flow** in [refine.md](refine.md#from-evidence-to-a-change) shows), then upload it and the synthetic set under the test key and score them there, with its model set to the other one. Bring the person those numbers, or ask them, before any change.

## Stages

### `brief_check`

First find the structure in use: `list_evaluations` with `assignment_mode: primary` and `limit: 1`, and read `version.structure_hash`. With no primary traffic yet (just after the build flow, say), use the newest imported example's (`assignment_mode: example`). With no evaluations at all, use the repo's decision file: the hash its upload answers ([Upload a file](tasks.md#upload-a-file)) is the structure in use, and `get_version` with it tells you whether it's registered. Without a file, use the newest registered version (`list_versions`, newest number first) and say so. With none of these, there is nothing to improve yet: tell the person and suggest the build flow. Read it with `get_version` (its `ref` is the hash or the version number). When the repo has no decision file (the structure lives in the app's code), write `moonbounce/<decision_id>.json` from it as `{decision_id, structure, objective}`, with the stored objective from `get_objective` (leave `objective` out when none is stored), and tell the person. When the repo has one, check it is the same structure: the decision file's upload must answer that hash. If they differ, ask the person which one is current.

Then read `moonbounce/<decision_id>.brief.md`. When there is no brief, draft one from the structure, in the sections [build.md](build.md#files) lists:

- each question's instructions and criteria become clauses;
- the result's outcomes become the Outcomes section.

Ask the person about every clause you had to infer.

Set or update the front matter: `flow: improve`, `stage: brief_check`.

Then check the brief against what the decision does and what people said:

- **Every clause is covered:** some question or rule applies it.
- **Every question traces to a clause:** a question with no clause is a gap in the brief or a question to drop.
- **Nothing contradicts it:** read recent `reasons` and `expected` from `list_cases` (newest first), and the feedback export (REST `GET /v1/decisions/{decision_id}/feedback`, not an MCP tool) when you have a key. Look for a reason or an `expected_outcome` that contradicts a clause.

Show the person the gaps and contradictions. A contradiction is settled in the brief first: the person decides whether the clause or the feedback is wrong.

- **The clause is wrong:** update the brief. The feedback stands.
- **The feedback is wrong:** correct the label, or later case runs keep scoring against it. With the person's confirmation, send `submit_feedback` about the case's `evaluation_id` with `source: "owner"`, an `expected_outcome` claim with the right outcome, and a `reason` naming the clause. The owner's label outranks reviewers' and imported ones, and a later owner label replaces an earlier one ([Add labeled data](tasks.md#add-labeled-data)). The outcome must be one the case's own structure can produce (a 400 otherwise). When the right outcome is one the brief added since, the case can't be relabeled in place and keeps scoring against its old label: tell the person.

Record the structure hash you checked against and the date in the front matter.

### `baseline`

The baseline is the structure `brief_check` found, unless the person names another.

If the baseline isn't registered (the repo's decision file, say), register it first by the hash its upload answered ([Register it](tasks.md#register-it)). Score it on the cases with `create_run` with `cases` and `loss_report`, with one run per label source: `cases.label_source` takes one value, so `human`, `unknown` and `synthetic` are three runs. Check `list_cases` with each `label_source` first and skip a source with no cases, since a case run that matches nothing is a 400 `NO_CASES`. A case run takes at most 200 cases: with more, page it (`cases.limit: 200`, then `cases.cursor` set to the last run's `cases.next_cursor`) and list every page's run on its side of `loss_report`. Keep the reports and run ids. The baseline is the first kept version, so it is what the first candidate must beat. If the objective isn't stored in this environment, or differs from the decision file's, store it first (`set_objective`).

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

Use fresh case writers and a fresh labeler, as in the build flow's [`cases`](build.md#cases) and [`labels`](build.md#labels) stages, with three differences: size the set to the cases above, not five a clause; there is no person's data to put in first; and each writer takes its own hundred of item ids above the highest `syn-` id already in the synthetic file, and their files merge into it, never into the build dataset. Write the result to `moonbounce/<decision_id>.synthetic.jsonl`, with `label_source: "synthetic"` (or `human` where the person settled the label). These lines stay in the repo. They are scored over the file's upload ([refine.md](refine.md#score-a-version)), apart from the real cases, and never imported among the real cases.

Done when you can say how many usable cases there are, how many were dropped and why, and what the synthetic set adds.

### `refine`

Run [the refine loop](refine.md). Backtest each candidate against the last kept version (at first, the baseline) on the same cases, with the runs a preview would make but kept so you can page them:

1. **The kept version**: `create_run` with `cases`, its registered hash in `structure_hash`, and `refresh` set to its questions that the candidate rewords or drops. Reuse its runs from the last comparison only when the cases, the `refresh` set and the kept version are all the same and the runs haven't expired. A different `refresh` set changes which answers the base replays, so score the base again.
2. **The candidate**: `create_run` with `cases`, by the `structure_hash` its upload answered ([Candidates](refine.md#candidates)), over the same cases.
3. **Judge** with `loss_report`, the kept version's runs as `base`.

With more than 200 cases, page both versions over the same pages, in the same order (the same filters, then each `next_cursor`), and list every page's run on its side. Pages are cut by label, not by version, so the two sides line up.

Run each step once per label source that has cases, as in `baseline`, and judge two reports:

- **The deciding report**: the `human` runs. When there are fewer than about 20 human-labeled cases, too few to tell, add the `unknown` runs to both sides of the same `loss_report` (it takes several `run_ids` a side) and tell the person which labels decided.
- **With no human-labeled or `unknown` cases at all** (only synthetic labels, just after the build flow for instance), the `synthetic` runs decide. Tell the person the decision has no real labels yet, and that review of real traffic is what will test it.
- **Reported apart**: the `synthetic` runs, and the `unknown` ones, whenever they didn't decide.

Score the synthetic set over its upload, a run per 200 lines, judged by its own `loss_report`. When the deciding report and the rest disagree, the deciding report wins.

Register each version you keep (`register_decision` with its `structure_hash` alone, while its upload or one of its runs lives), so it can be the next base. `create_preview` gives the same comparison in one call when you only need the headline: it has no run ids to page and doesn't split by label source.

### `red_team`

The stored cases cover only what traffic has sent so far, and a reworded question or a moved cut can open a weakness no case touches. Once the person says `refine` is good enough, a red team attacks what the settled version changed, as in the build flow's [`red_team`](build.md#red_team):

1. **Write.** Brief a fresh red team with the brief, the settled structure, and the diff from the baseline: "Write 10 to 30 cases this change is likely to get wrong, whose right outcome the brief settles: paraphrases near a moved cut or threshold, cases that hinge on a reworded question's criteria, near-misses for a new question, and cases the old version got right that the change could break. Use only the inputs, types and ranges the brief lists. Work only inside the repo. Write files with your file tool, never with a heredoc or `echo`; a `cp` or a `jq` to a file under `moonbounce/` is fine, but never onto a file the command reads. Run one plain command at a time: no `cd`, `;` or `&&` chains, loops, shell variables, `$(…)` or heredocs. Never put the person's data or item ids in a command." It writes cases and intended labels apart, with `metadata.origin: "red"`. Any model can be the red team; when the person has another model or vendor available, recommend using it, say that it sends the brief, the structure and the diff to that vendor, use it only on their explicit yes, and record which model wrote each round.
2. **Label.** The labeler labels the round blind, on a copy without `metadata` (`jq -c 'del(.metadata)'`): its attack names hint at the answer. Take disagreements and unsure cases to the person.
3. **Keep.** Add the round, with its settled labels, to the end of `synthetic.jsonl`, so later candidates are scored on it, and upload it again.
4. **Score.** Score the settled version once over the round's lines (`items_from` with `offset` the file's line count before them), and report it by attack.
5. **Next: a Stop.** Show the round by attack, then recommend and let the person choose:
   - **Back to `refine`**, recommended when the round broke a mistake the objective rates costly or severe (look up its cell's severity in the decision file's `objective`, not the report's outcome-weighted row `loss`). Read the right cases nearest each broken one (`like`) before the fix, then attack the version that comes out of it. On this return, a candidate that fixes red cases may leave the real-label loss unchanged, which `loss_report` doesn't accept: keep it by the [red-team repair rule](refine.md#keep-or-drop-a-change), and tell the person that is the rule you used.
   - **Another red-team round** on the same version, while fewer than three have run.
   - **On to `ship`**, recommended when the round broke nothing worse than a normal mistake, or after three rounds.

   Whatever they choose, the round's cases stay in the synthetic set.

Red cases are synthetic: they are reported apart and never outweigh real labels in deciding a keep. One rule still applies to them: a candidate that makes a new severe mistake on any label, red cases included, is dropped ([refine.md](refine.md#keep-or-drop-a-change)). So a round can block a dangerous change but can't carry one through.

### `ship`

The person decides to ship. Then do these in order:

1. Register the chosen structure in each environment it will run in (`register_decision`), and check the returned `version.structure_hash` against the one you scored.
2. Check the model in effect (`get_model`), and store the decision file's objective (`set_objective`; `unknown_outcomes` must be empty).
3. Save the chosen structure in the decision file, and commit it only when the brief's `commits` names a branch. Then offer to change the app to send the new hash ([Wire it into the app](tasks.md#wire-it-into-the-app)), which stops for the person before touching their code. Evaluations start recording the new hash once the app sends it.

Set the brief's `structure_hash` to the shipped hash and `stage` to `done`.

Spot checks are the measure of what was shipped: a random sample of production evaluations that the person's reviewers label in Moonbounce's Review app. They happen only when the team reviews, so tell the person that without reviews there is no measure after shipping. The next run of the improve flow reads them as cases.

## Going back

- **A policy change or a contradiction found at `refine`:** go back to `brief_check`. Then re-run `baseline`, because the objective or the labels may have moved.
- **New feedback arriving mid-loop:** re-run `cases` before the next candidate.
