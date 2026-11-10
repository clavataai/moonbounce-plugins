# The improve flow: a decision in use

Use this as a library of stage procedures selected by the stored advisory plan and [intake adapter](intake.md). The adapter controls transitions, limits and effects; do not run this page as an independent workflow.


The decision exists and real feedback is the main signal. There is no "shipped version" pointer: each evaluation records whichever version ran.

| Stage | Does | Leaves | Done when |
|---|---|---|---|
| `brief_check` | Checks the selected version, brief and feedback for relevant gaps; assess reports discrepancies without editing the candidate or settings | Scoped brief findings, selected version and evidence references | The requested inspection is reported; any correction follows the selected operation and authorization |
| `baseline` | Pins the user-selected version, objective and model; obtains or reuses measurement appropriate to the requested scope | Baseline loss report | A verified baseline report exists, or the blocked/unmeasured parts are explicitly reported |
| `cases` | Audits available labeled cases and prepares only selected gaps or regression cases, preserving provenance and independent groups | Case list, synthetic JSONL | The usable cases are known, with drop reasons |
| `refine` | The refine loop, backtesting each candidate against the last kept version on the labeled cases | Kept local candidates and comparison reports; registration only when selected and authorized | The requested stopping point or remaining allowance is reached, with kept/rejected changes reported |
| `red_team` | A red team (another model when the user has one) attacks what the settled version changed; the round is labeled blind, settled with the user, scored once and joins the synthetic set | `work/red-<n>.jsonl`, synthetic additions | The scoped attack report is complete; further repair, attacks or deployment follow the user's request |
| `ship` | Registers the selected version when requested; application wiring and deployment are separate explicit work | Decision file, registered hash | The requested registration is verified; no application change follows automatically |

At a **Stop**, show the person what the stage left and wait for their answer before moving on. When the question is how a case should go or which clause wins, ask what the person would decide before showing your recommendation, as the [Stop format](../SKILL.md#what-the-person-sees) says. Record completed work and the currently selected procedure in the brief with its current date, and commit only when the brief's `commits` names a branch.

The brief is context here. When it exists, it settles which side is right when feedback conflicts, and a policy change updates it first. Reports show human-labeled and synthetic cases apart, and real data decides when they disagree.

Use the key for the environment whose traffic you are improving, usually production. Scoring is always dry runs, which store no evaluation; each run keeps its results and labels for about a day, then they are gone, as uploads are. Beyond those runs and uploads, selected and authorized create/revise work can also write: registering a version (`register_decision`, only when selected use or requested objective setup requires it), storing the objective (`set_objective`, before the first scoring when the stored one differs from the decision file's, and at `ship`), storing the description (`set_description`, at `brief_check`, or at `baseline` when the decision isn't registered until then), a label the person has you correct (at `brief_check`, or a ruling on a stored synthetic case in `refine`: [Check before you fit](refine.md#check-before-you-fit)), and `ship`. Registering changes nothing the decision runs.

Two writes change what a decision in use does, so each is a **Stop**: say what it changes, and wait for the person's yes.

- **The objective** (`set_objective`) changes what every later report and suggestion scores by.
- **The model** (`set_model`) changes what the decision's live evaluations run on in that environment, from the next call, whatever version the app sends. This flow never changes it on its own. Runs use the environment's model setting, and a run takes no model of its own, so a comparison needs a place with its own setting: the Test environment, which needs an `env:test` key (sign-in reaches Production only). Cases and uploads are per environment, so export the production cases to a file first (`list_cases` with `include=content`, written as **A new attribute in the improve flow** in [refine.md](refine.md#from-evidence-to-a-change) shows), then upload it and the synthetic set under the test key and score them there, with its model set to the other one. Bring the person those numbers, or ask them, before any change.

## Stages

### `brief_check`

When resuming after an initial measurement, use the current request, stored plan and verified evidence to select the next work. A local dataset can be scored through an items upload; import it only when example storage is requested. Resume an unfinished import only when import is selected. Report the confidence and coverage of available labels, including synthetic ones, without requiring a fixed human-label count, a full-build restart or an automatic import before useful work can continue.

First find the version in use: `list_evaluations` with `assignment_mode: primary` and `limit: 1`, and read `version.structure_hash`. With no primary traffic yet (just after the build flow, say), use the newest imported example's (`assignment_mode: example`). With no evaluations at all, use the repo's decision file: the hash its upload answers ([Upload a file](tasks.md#upload-a-file)) is the version in use, and `get_version` with it tells you whether it's registered. Without a file, use the newest registered version (`list_versions`, newest number first) and say so. With none of these, there is nothing to improve yet: tell the person and suggest the build flow. Read it with `get_version` (its `ref` is the hash or the version number). When the repo has no decision file (the decision lives in the app's code), write `moonbounce/<decision_id>.json` from it as `{decision_id, structure, objective}`, with the stored objective from `get_objective` (leave `objective` out when none is stored), and tell the person. When the repo has one, check it is the same version: the decision file's upload must answer that hash. If they differ, ask the person which one is current.

Then read `moonbounce/<decision_id>.brief.md`. When there is no brief, draft one from the decision, in the sections [A decision's files](files.md) lists for the brief:

- each question's instructions and criteria become clauses;
- the result's outcomes become the Outcomes section.

Record inferred clauses and their basis. Clarify only a consequential ambiguity or required ruling; supported assumptions can remain visible without asking the person to restate every inferred clause.

When this procedure is selected, record its execution context in the front matter: `flow: improve`, `stage: brief_check`. Front matter does not select the next procedure or override the current request and plan.

When the decision has no description (`get_decision` reports `description: null`, or the decision isn't registered yet), or a policy change the person confirms makes it wrong, write the brief's Description sentence and confirm it with the person at this stage's Stop. Store it once the decision exists only for selected, authorized live create/revise work ([Describe the decision](tasks.md#describe-the-decision)). Assess and draft-only plans report a needed description change without storing it.

Then check the brief against what the decision does and what people said:

- **Every clause is covered:** some question or rule applies it.
- **Every question traces to a clause:** a question with no clause is a gap in the brief or a question to drop.
- **Nothing contradicts it:** read recent `reasons` and `expected` from `list_cases` (newest first), and the feedback export (REST `GET /v1/decisions/{decision_id}/feedback`, not an MCP tool) when you have a key. Look for a reason or an `expected_outcome` that contradicts a clause.

Show the person the gaps and contradictions. A contradiction is settled in the brief first: the person decides whether the clause or the feedback is wrong.

- **The clause is wrong:** update the brief. The feedback stands.
- **The feedback is wrong:** correct the label, or later case runs keep scoring against it. With the person's confirmation, send `submit_feedback` about the case's `evaluation_id` with `source: "owner"`, an `expected_outcome` claim with the right outcome, and a `reason` naming the clause. The owner's label outranks reviewers' and imported ones, and a later owner label replaces an earlier one ([Add labeled data](tasks.md#add-labeled-data)). The outcome must be one the version it was stored under can produce (a 400 otherwise). When the right outcome is one the brief added since, the case can't be relabeled in place and keeps scoring against its old label: tell the person.

Record the version hash you checked against (`structure_hash`) and the date in the front matter.

### `baseline`

Under an intake plan, assess and draft-only work registers nothing and changes no model, objective or description settings. A missing registration or different stored objective during assessment is a conflict to report; use only measurement compatible with the pinned candidate and objective.

The baseline is the version `brief_check` found, unless the person names another.

For selected, authorized live create/revise work, register an unregistered baseline first by the hash its upload answered ([Register it](tasks.md#register-it)). Score it on the cases with `create_run` with `cases` and `loss_report`, with one run per label source: `cases.label_source` takes one value, so `human`, `unknown` and `synthetic` are three runs. Check `list_cases` with each `label_source` first and skip a source with no cases, since a case run that matches nothing is a 400 `NO_CASES`. A case run takes at most 200 cases: with more, page it (`cases.limit: 200`, then `cases.cursor` set to the last run's `cases.next_cursor`) and list every page's run on its side of `loss_report`, which takes at most 50 runs a side, resent failures included: with more cases than that holds, score the newest pages that fit, the same pages for every version, and say so. Keep the reports and run ids. The baseline is the first kept version, so it is what the first candidate must beat. For that authorized create/revise setup, store the objective first if missing or different (`set_objective`), and the description when `brief_check` wrote one before the decision existed (`set_description`). Assess reports either mismatch without applying these writes.

### `cases`

List the labeled cases with `list_cases`, whatever version they ran under. Set `outcomes` to the outcomes either version can produce.

Read `excluded`: it says which evaluations with feedback aren't cases, and why. Usable cases have both of these:

- stored content;
- an outcome label that either version's result can produce. A candidate that drops an outcome is wrong on its cases, not excused from them.

Disagreements drive changes and agreements guard against regressions, so both stay in the backtest.

Then add targeted synthesis where real cases are thin:

- rare outcomes;
- edge cases the brief names that no case covers;
- new policy clauses;
- regression probes for a change you plan.

Write it as [Writing cases](cases.md) says, fresh case writers each labeling their own cases, with three differences: size the set to the selected gaps and allocated allowance; there is no person's data to put in first; and each writer takes its own hundred of item ids above the highest `syn-` id already in the synthetic file, and their files merge into it, never into the build dataset. Write the result to `moonbounce/<decision_id>.synthetic.jsonl`, with `label_source: "synthetic"` (or `human` where the person settled the label). Those labels are checked in `refine`, before a candidate is changed to fit them ([Check before you fit](refine.md#check-before-you-fit)). These lines stay in the repo. They are scored over the file's upload ([refine.md](refine.md#score-a-version)), apart from the real cases, and never imported among the real cases.

Done when you can say how many usable cases there are, how many were dropped and why, and what the synthetic set adds.

### `refine`

Run [the refine loop](refine.md). Backtest each candidate against the last kept version (at first, the baseline) on the same cases, with the runs a preview would make but kept so you can page them:

1. **The kept version**: `create_run` with `cases`, its registered hash in `structure_hash`, and `refresh` set to its questions that the candidate rewords or drops. Reuse its runs from the last comparison only when the cases, the `refresh` set and the kept version are all the same and the runs haven't expired. A different `refresh` set changes which answers the base replays, so score the base again.
2. **The candidate**: `create_run` with `cases`, by the `structure_hash` its upload answered ([Candidates](refine.md#candidates)), over the same cases.
3. **Judge** with `loss_report`, the kept version's runs as `base`.

With more than 200 cases, page both versions over the same pages, in the same order (the same filters, then each `next_cursor`), and list every page's run on its side, within the 50 runs a side `baseline` keeps to. Pages are cut by label, not by version, so the two sides line up.

Run each step once per label source that has cases, as in `baseline`, and judge two reports:

- **The deciding report**: the `human` runs. When there are fewer than about 20 human-labeled cases, too few to tell, add the `unknown` runs to both sides of the same `loss_report` (it takes several `run_ids` a side, at most 50: the human and `unknown` pages together, resent failures included, keep within that) and tell the person which labels decided.
- **With no human-labeled or `unknown` cases at all** (only synthetic labels, just after the build flow for instance), the `synthetic` runs decide. Tell the person the decision has no real labels yet, and that review of real traffic is what will test it.
- **Reported apart**: the `synthetic` runs, and the `unknown` ones, whenever they didn't decide.

Score the synthetic set over its upload, a run per 200 lines, judged by its own `loss_report`. When the deciding report and the rest disagree, the deciding report wins.

Register a kept version only when the selected authorized use requires it (`register_decision` with its live structure hash); a kept local candidate and a deployed version are distinct. `create_preview` gives the same comparison in one call when you only need the headline: it has no run ids to page and doesn't split by label source.

### `red_team`

The stored cases cover only what traffic has sent so far, and a reworded question or a moved cut can open a weakness no case touches. When targeted attacks are selected, a red team examines the pinned candidate or requested revision diff ([A red-team round](red-team.md) has what the flows share):

1. **Write.** Brief a fresh red team with the brief, the settled version, and the diff from the baseline: "Write 10 to 30 cases this change is likely to get wrong, whose right outcome the brief settles: paraphrases near a moved cut or threshold, cases that hinge on a reworded question's criteria, near-misses for a new question, and cases the old version got right that the change could break. Use only the inputs, types and ranges the brief lists. Work only inside the repo. Write files with your file tool, never with a heredoc or `echo`; a `cp` or a `jq` to a file under `moonbounce/` is fine, but never onto a file the command reads. Run one plain command at a time: no `cd`, `;` or `&&` chains, loops, shell variables, `$(…)` or heredocs. Never put the person's data or item ids in a command." It writes cases and intended labels apart, with `metadata.origin: "red"`. A round of 15 or more cases on more than three clauses goes to two or three writers at once, each on a share of the clauses the change touches ([Splitting the round](red-team.md#splitting-the-round)). Who writes it, another model included: [Who writes it](red-team.md#who-writes-it).
2. **Label.** Settle the round's labels as [Settling its labels](red-team.md#settling-its-labels) says.
3. **Keep.** Add the round to the end of `synthetic.jsonl`, so later candidates are scored on it, and record it, as [Keeping it](red-team.md#keeping-it) says.
4. **Score.** Score the settled version once over the round's lines (`items_from` with `offset` the file's line count before them), and report it by attack.
5. **Report and stop where requested.** Report results by attack and the relevant loss severity, with provenance and limitations. Recommend useful follow-up work without automatically selecting repair, another round or deployment. If the user requests repair, use the requested scope and allowances, read the nearby right cases before fitting, and apply the [red-team repair rule](refine.md#keep-or-drop-a-change) when appropriate, disclosing that rule. An additional attack or application change is selected separately.

Red cases are synthetic: they are reported apart and never outweigh real labels in deciding a keep. One rule still applies to them: a candidate that makes a new severe mistake on any label, red cases included, is dropped ([refine.md](refine.md#keep-or-drop-a-change)). So a round can block a dangerous change but can't carry one through.

### `ship`

Select the requested effects separately; this section is not a mandatory continuation after scoring or an attack report. Assessment does not select any of these writes:

1. **Selected registration.** Register the chosen version in the requested environment (`register_decision`), and check the returned `version.structure_hash` against the one you scored.
2. **Selected live setup.** Check the model in effect (`get_model`), and store the objective only when that setup is requested and authorized (`set_objective`; `unknown_outcomes` must be empty).
3. Save the chosen version in the decision file, and commit it only when the brief's `commits` names a branch. Application changes are a separate explicit request; when selected, put the version in the app ([Wire it into the app](tasks.md#wire-it-into-the-app)), which stops for the person before touching their code: an app that sends the decision file inline needs only a deploy, and one that sends a hash needs the new hash. Evaluations start recording the new hash once the app runs it.

Record the verified registered hash and completion of the selected work. Registration alone does not mean the application deployed or began using that version.

Spot checks are the measure of what was shipped: a random sample of production evaluations that the person's reviewers label in Moonbounce's Review app. They happen only when the team reviews, so tell the person that without reviews there is no measure after shipping. The next run of the improve flow reads them as cases.

## Going back

These are evidence-invalidation triggers, not mandatory transitions. Reconcile the current request, valid receipts and remaining limits before selecting affected procedures. Assessment reports the consequences without repairing the candidate or changing settings.

- **Changed policy or a contradiction:** inspect the affected brief, labels and comparisons. Select repairs and new measurement only when needed for the current request; assessment reports incompatibilities without editing the candidate or settings.
- **New feedback during requested refinement:** verify its relevance and label eligibility before a further comparison. Reuse the unaffected comparison set; do not restart unrelated work or expand the requested scope automatically.
