# The build flow: a new decision from a policy

The person has a policy and some data, or none. The brief is where the first structure comes from. The dataset lives in the caller's repo, versioned by git. Scoring sends its items to Moonbounce as dry runs, which keep the run (answers, labels and attribute values) for about a day so `loss_report` can judge it; only the last stage, `import`, stores the dataset for good.

| Stage | Does | Leaves | Done when |
|---|---|---|---|
| `objective` | Names the outcomes, then asks which wrong answer is worse, in plain words; writes the objective | Objective in the decision file | **Stop:** the user confirms the outcomes, severities and fallback |
| `brief` | Writes the brief from the policy and the conversation | `moonbounce/<id>.brief.md` | **Stop:** the user confirms it |
| `cases` | A case writer fills the coverage grid, on top of the user's own data | `moonbounce/<id>.examples.jsonl` | No empty grid cell, no near-duplicates |
| `labels` | A labeler labels from the brief alone; disagreements and a sample of agreements go to the user; brief and labels are revised until they agree | Labeled JSONL, brief edits | **Stop:** the user accepts the labels |
| `structure` | The author writes the first structure from the brief | Decision file structure | It validates, and every brief clause maps to a question |
| `refine` | The refine loop over the dataset | Accepted versions, loss reports | **Stop:** the user says it's good enough |
| `red_team` | A red team (another model when the user has one) attacks the version `refine` settled on; the round is labeled blind, settled with the user, scored once and joins the dataset | `work/red-<n>.jsonl`, dataset additions | **Stop:** a round finds nothing wrong, or after three rounds. A round that breaks the version goes back to `refine` first, then attacks the new version |
| `blind_check` | An agent that has seen neither the structure nor the examples writes a fresh set from the frozen brief, by the same grid; scored once | `moonbounce/<id>.blind.jsonl` | **Stop:** the user accepts the result. Failures join the training set and the flow returns to `refine`, then `red_team`; the next freeze gets a fresh blind set |
| `import` | With the person's go-ahead, `import_examples` stores the dataset and runs the engines, so previews and the improve flow can use it | Server copy of the dataset | Every example stored, except ones the person agreed to leave out (a retryable failure sent again in a new call; one with `retryable: false` fixed, dropped or taken to the person), and `loss_report` over the runs matches the last scores; the build flow ends |

At a **Stop**, show the person what the stage left and wait for their answer before moving on. When a stage is done, set the brief's `stage` to the next one and its `date` to today, and commit only when the brief's `commits` names a branch.

## Why a team

One agent that writes the cases, labels them, writes the structure and scores it will pass its own test: it imagines a world, then fits it. Split the jobs across fresh subagents instead, so the structure is tested against cases its author never shaped. The barriers are instructions and paths, not access control. They hold only if every role is a fresh subagent briefed with the paths it may read. A fork inherits your context, so no role runs as a fork.

| Role | Job | Reads | Writes |
| --- | --- | --- | --- |
| **You** (orchestrator) | Talk with the person, write the objective and brief, route paths, compare labels, score versions, keep the front matter | Everything | The decision file's `objective`, the brief, the labeled JSONL |
| **Case writer** | Fills the coverage grid around the person's data, keeping its own intended labels out of the cases | The brief, the person's data with its labels | `examples.jsonl` (unlabeled), `work/intended.jsonl` |
| **Labeler** | Labels every case from the brief alone | The brief, one unlabeled file | `work/labels.jsonl` |
| **Author** | Writes and revises the structure | The brief, the labeled `examples.jsonl`, loss reports you give it | The decision file's `structure`, `work/notes.md` |
| **Red team** | Writes cases meant to break the version `refine` settled on, at `red_team` | The brief, the decision file, `examples.jsonl` | `work/red-<n>.jsonl`, `work/red-<n>.intended.jsonl` |
| **Blind writer** | Writes the blind set from the frozen brief | The brief and the grid only | `blind.jsonl` (unlabeled), `work/blind-intended.jsonl` |

`work/` is `moonbounce/work/<decision_id>/`. It stays in the repo with the rest, so the flow resumes from the files alone. Never give the author the blind set, and never give the blind writer the structure or `examples.jsonl`.

## Files

- **The decision file**, `moonbounce/<decision_id>.json`: `{decision_id, structure, objective}`. A decision file already in the repo with no stored evaluations is a draft: the author starts from it at `structure`.
- **The brief**, `moonbounce/<decision_id>.brief.md`. It holds the flow's front matter (see the skill's [Where a flow stands](../SKILL.md#where-a-flow-stands)) and these sections:
  - **Policy**: the policy, in the person's words or quoted from its source.
  - **Outcomes**: each outcome with a one-line definition, and what `escalate` or an abstention means.
  - **Mistakes**: why the objective's severities were chosen.
  - **Inputs**: the `state` keys and attribute values the app sends, and which can be missing.
  - **Scope**: what is in scope and what is out.
  - **Edge cases**: known edge cases.
  - **Clauses**: a numbered list of the rules the decision applies.
- **The dataset**, `moonbounce/<decision_id>.examples.jsonl`. Each line is exactly one `import_examples` example: `{item_id, state, attribute_values?, metadata, claims, label_source, reason?}`.
  - `metadata` carries `cell` (the grid cell), `group` and `origin` (`user`, `synthetic` or `red`).
  - `group` is bookkeeping for you and the case writer: paraphrase twins with the same outcome share one, and a boundary pair whose outcomes differ (the same message at day 29 and day 31) gets two. The server doesn't read it: it groups imported cases by their content and text.
  - Before `labels`, lines have no `claims` or `label_source`.
  - `label_source` is `human` for a label the person gave or settled, `synthetic` for one the case writer and the labeler agreed on, and `unknown` for the person's data when nobody knows who labeled it.
  - `import_examples` takes the lines as they are.
- **The blind set**, `moonbounce/<decision_id>.blind.jsonl`: the same format, in its own file, labeled at `blind_check`.
- **Run ids**, `work/runs.json`: for each scoring, the structure hash, the file scored (with its commit when there is one), the `run_id`s and when they expire. Runs expire after about a day, so a resumed flow re-scores what it needs.

## The coverage grid

The case writer and the blind writer fill the same grid:

- each criterion on and off, and in combination;
- cases near the line, where one fact decides the outcome;
- borderline wording;
- phrasing and form: indirect, multilingual, long or short, irrelevant detail, missing or contradictory fields (only those the brief says can be missing);
- adversarial cases, where the keywords point one way and the facts the other;
- every outcome, `abstain` and `escalate` included.

Give each cell at least two cases. Each criterion's on/off and combination cells come from the brief's clauses.

## Stages

### `objective`

Create the brief with its front matter (`flow: build`, `stage: objective`) and the decision file if it doesn't exist (`{decision_id, objective}`).

First name the outcomes: agree with the person on the exact names the decision returns (`grant`, `escalate`, `deny`, say), each with a one-line meaning. They go in the brief's Outcomes section, and every label and severity uses them exactly.

Then tell the person what their answers are for, before asking. The refine loop keeps the version with the lowest **loss**: every wrong answer adds a weight, and the loss is their total. The weights are theirs to choose, and they should come from their use case: what each kind of wrong answer costs whoever lives with it. Only the ratios matter. Money is one way to think about it, as an example, not a figure to supply: if wrongly denying a refund loses a customer worth about $80, and a needless escalation costs about $4 of an agent's time, the denial should weigh about twenty times the escalation. Harm, time, risk or trust work the same way.

Then ask which wrong answer is worse, in plain words:

- **Two outcomes:** this is precision against recall. Ask: "Would you rather catch every <outcome>, even with some false alarms, or only flag <outcome> when you're sure?" Write `{"outcome": "<outcome>", "priority": "catch_all"}`, `"be_sure"`, or `"balanced"` when neither matters more.
- **Three or more:** ask once per right answer: "When the right answer is `grant`, how bad is answering `deny`? And `escalate`?", each as barely matters, a normal mistake, costly, or unacceptable. Write them as `mistakes` with severity `minor`, `normal`, `costly` or `severe`: weights of 0.5, 1, 3 and 10. The person can say "the rest are normal" at any point: a mistake not listed is normal. When they think in ratios ("about twenty times worse"), write a weight directly as `loss`, on the scale where an ordinary mistake is 1. A weight of 10 or more counts as severe, and the loop drops any candidate that adds a severe mistake on a human label, so keep that for the mistakes they would never accept. Tell them the weights you wrote.

Ask whether one outcome is the cheap place to land when unsure (often `escalate`): that is `fallback`, and sending a case there wrongly counts as minor. It only weighs mistakes; how the decision abstains or escalates is set in the structure.

You can ask whether they want a target; it is optional, and soft. The refine loop runs until the person says the decision is good enough, and a target never stops it on its own: reports say whether it is met (`target_met`), as a reference point. Loss per case counts each outcome equally, however many cases it has, so a target reads per outcome: `{"max_loss_per_case": 0.1}` is about one normal mistake in every ten cases of each outcome, on average across outcomes, and `0.05` one in twenty. A severe mistake counts as ten normal ones on the same outcome. If they want one, ask how often a normal mistake is acceptable for a typical outcome, write the matching target, and say what you chose and what it means.

Write the answers into the decision file's `objective` ([Set the objective](tasks.md#set-the-objective)), and the why into the brief's Mistakes section. You store the objective on the server at `structure`, once the decision exists.

### `brief`

Write the brief from the policy and the conversation. Number the clauses. Ask the person about every gap rather than guessing: an unclear clause here becomes a disagreement at `labels`.

When a clause depends on a fact the app already has (an amount, a count, days, a flag, a category), you can raise here making it an attribute rather than having the engine judge it ([Facts are attributes](design.md#facts-are-attributes-judgment-is-a-question)), and note in the Inputs what the app would send. It doesn't have to be settled now: the author or the refine loop can propose it later.

When the person has labeled data already, read it, and its reasons, before you write the clauses. Each reason is the policy as someone applied it, so it should map to a clause. Act on what doesn't:

- **A reason no clause covers:** a missing clause. Draft it and ask the person.
- **A reason that contradicts a clause:** ask the person which is right, and settle it in the brief before they confirm it.

### `cases`

Give the case writer the person's data as it is, labels, `label_source` and reasons included. Their labels are how the person actually decides, and the case writer needs them to write cases consistent with that. What it keeps out of the cases is its own intended labels; the labeler is the one that never sees a label.

Brief a fresh case writer: "Write cases for the decision in `<brief>`. Use the person's data in `<files>` first, with `metadata.origin: "user"`, turned into the `state` the brief says the app sends (a ticket's `text` may be the app's `state.message`, for instance); mark everything else `synthetic`. Fill every cell of the coverage grid with at least two cases, with paraphrase twins that have the same outcome sharing `metadata.group`, and a boundary pair (one fact apart, different outcomes) in two groups. Use only the `state` keys and attribute values the brief says the app sends. Write each case to `<examples>` as `{item_id, state, attribute_values?, metadata: {cell, group, origin}}`. Write its intended outcome and a one-line reason citing a clause to `<intended>` as `{item_id, label, reason}`, never in the case itself. The person's data carries their labels: treat them as precedent. Write none for their lines, give a synthetic case close to one of their lines the same label, and where the brief would label such a case otherwise, list both item ids in `<intended>` with `label: null` and a reason naming the conflict, instead of choosing. Read nothing else."

When the person's data carries labels, those lines go into the dataset with their label, `label_source` and `reason` as they are. Use `label_source: "human"` only when the person says they or their team labeled it, and `unknown` when nobody knows who did. Their label is the intended one, so the case writer writes no intended label for them. The labeler still labels them blind. Take each conflict the case writer listed to the person at `labels`, with the disagreements.

Done when no grid cell is empty and no two cases are near-duplicates. Near-identical items join one group on import, so a duplicate adds nothing.

### `labels`

Give the labeler a copy without labels or `metadata` (its cell names and origins hint at the answer), so it labels every line blind, the person's labeled lines included: `jq -c 'del(.claims, .label_source, .reason, .unsure, .metadata)' moonbounce/<decision_id>.examples.jsonl > moonbounce/work/<decision_id>/unlabeled.jsonl`.

Brief a fresh labeler: "Label each case in `<unlabeled>` using only `<brief>`. For each, give the outcome, whether you are unsure, and a one-line reason citing a clause. When the brief doesn't settle a case, mark it unsure and say what the brief is missing. Write `<labels>` as `{item_id, label, unsure, reason}`."

Compare the labeler's labels with the intended ones, and take these to the person:

- every disagreement;
- every case the labeler marked unsure;
- a sample of agreements, about one in ten and at least five.

Same-model writers and labelers rarely disagree outright, so the unsure cases carry most of the signal.

When the labeler disagrees with one of the person's own labels, show the person their reason beside the labeler's. Either the brief misses what their reason says, and the brief changes, or the old label was wrong, and they relabel it. Their reason stays with the label unless they change the label.

Each disagreement is ambiguity in the brief, not noise. Settle it with the person, update the brief or drop the case, and never average two labels. When the brief changes, both the case writer and the labeler re-check the labels they wrote before the change.

Write the settled labels into `examples.jsonl`:

- `claims: [{kind: "expected_outcome", value}]`;
- `label_source`: `human` when the person gave or settled the label, `synthetic` when the two agents agreed;
- `reason`.

Every label must be an outcome the brief's Outcomes section names, so the structure can produce it.

### `structure`

Brief a fresh author: "Read `<design guide>` first. Then write the structure for the decision in `<brief>` at `<decision file>`, using `<examples>`. Make facts attributes and judgments questions: use the attribute values the brief's Inputs list, and in `<notes>` propose any other fact in the state you would make an attribute. State the brief's criteria in each question rather than wording drawn from particular cases. Read only paths the brief says the app always sends. Map every clause in the brief to the question or rule that applies it, in `<notes>`. Validate with `hash_structure`."

When the author proposes making a fact in the state an attribute that the brief's Inputs don't list, ask the person first: today the app must send it in `attribute_values`. With a yes, add it to the Inputs and copy the values into the dataset's lines ([Facts are attributes](design.md#facts-are-attributes-judgment-is-a-question)); with a no, the author keeps a question.

Then register it (`register_structure`) in the environment the decision will run in. The first registration creates the decision and its binding from the deployment's default. Then do these in order:

1. Set the binding (`set_binding`) to the engine and model production will run. When the decision already has evaluations in this environment, it is in use: a binding change moves its live traffic, so stop and ask the person first.
2. Store the objective (`set_objective`), and check that `unknown_outcomes` is empty.
3. Run one dry-run `evaluate` on one case.

Put the structure's hash in the brief's front matter.

Done when it validates and every clause maps to a question.

### `refine`

Run [the refine loop](refine.md) over `examples.jsonl`, with the author editing and you scoring. The version registered at `structure` is the first kept version. Don't use `preview` here: it replays stored cases, and the build flow has none yet.

The stage stops when the person says the decision is good enough. Report `target_met` when they set a target, but it never stops the loop on its own.

### `red_team`

The dataset was written before the structure, and every version so far was tuned on it. Once the person says `refine` is good enough, a red team attacks that settled version: reasonable cases aimed at the weaknesses a reader of this structure can see. It finds what the dataset doesn't cover, and its cases then join the dataset, so later versions can't reopen those holes.

1. **Write.** Brief a fresh red team: "Read the decision in `<brief>` and its current structure in `<decision file>`, and the dataset in `<examples>`. Write 15 to 30 cases this structure is likely to get wrong, whose right outcome the brief settles. Aim for:
   - paraphrases of cases it gets right;
   - values just either side of a cut or threshold;
   - fields the brief says can be missing;
   - keyword traps, where the words point one way and the facts the other;
   - messages that mix two issues;
   - requests just out of scope.

   Every case must be one the app could really send, using only the inputs the brief lists. Write each case to `<red cases>` as `{item_id: "red-<n>-<k>", state, attribute_values?, metadata: {cell: <attack>, group, origin: "red"}}`. Write its attack, the outcome the brief requires and why to `<red intended>`."

   Any model can be the red team, the author's included. When the person has another model or vendor available (another agent's CLI, such as `codex exec -s read-only`), recommend using it: agents on the same model share blind spots, and another model finds weaknesses the author's can't see. Say that it sends the brief, the structure and the dataset (the person's own data included) to that vendor, and use it only on their explicit yes. Record which model wrote each round. A read-only CLI can't write files: ask it to print one JSONL line per case, with the attack, intended label and reason under an `intended` key. Capture its output and split it into the two files yourself.
2. **Label.** The labeler labels the round blind, from the brief alone, on a copy without `metadata` (its attack names hint at the answer): `jq -c 'del(.metadata)' <red cases> > <unlabeled red>`. Take disagreements and unsure cases to the person, as at `labels`. The red team's intended labels are one side of each comparison, never the answer. A disagreement that changes the brief goes back to `brief`, as any brief change does.
3. **Score.** Score the settled version on the round once, and show the person the result by attack.
4. **Keep.** Add the round's cases, with their settled labels, to `examples.jsonl`, and add the run from step 3 to the settled version's base runs.
5. **Next.** When the round broke something, go back to `refine` to fix it, with the round now in the dataset, and attack the version that comes out of it. Stop and ask the person when a round finds nothing wrong, or after three rounds; then go on to `blind_check`.

### `blind_check`

First freeze the brief: record a sha256 of its text below the front matter as `frozen` (`awk 'n>=2; /^---$/{n++}' moonbounce/<decision_id>.brief.md | shasum -a 256`). The front matter changes as stages pass; the text below it is what the blind writer reads. Before scoring the blind set, check the hash still matches: if the brief changed, the blind set no longer tests it.

Brief a fresh blind writer that has seen neither the structure nor `examples.jsonl`: "Write cases for the decision in `<brief>`, filling every cell of this grid: `<grid>`. Write each case to `<blind>` as `{item_id, state, attribute_values?, metadata: {cell, group, origin: "blind"}}`, using only the `state` keys and attribute values the brief's Inputs section lists, and its intended outcome with a one-line reason citing a clause to `<blind-intended>` as `{item_id, label, reason}`, never in the case itself. Read nothing else."

Then:

1. Have a fresh labeler label them blind, on a copy without `metadata` (`jq -c 'del(.metadata)'`, as at `red_team`), and settle disagreements with the person as at `labels`. A settlement that changes the brief unfreezes it: go back to `labels` for the dataset.
2. Write the settled labels into `blind.jsonl` as at `labels`: `claims`, `label_source` and `reason`.
3. Score the last kept version on the blind set once, with one dry run and one `loss_report`.

Show the person the loss per case and the wrong cases, beside the dataset's numbers. If they accept the result, go on to `import`. If not, move the failures into `examples.jsonl` and go back to `refine`, then `red_team` again: the version that comes out is new, so it is attacked before it is checked. The next freeze gets a fresh blind set from a fresh blind writer.

Never revise against the blind set, and never show it to the author.

### `import`

**Stop** first: tell the person what the import does (it stores every example as an evaluation in this environment, runs the engines once, and makes the labels the decision's cases for previews and the improve flow), what it will cost (about one dry run of the dataset), and wait for their go-ahead.

Store the dataset and the blind set, each with its own import: `import_examples` with up to 200 examples a call, the file's lines as they are (`jq -cs '_nwise(200)' moonbounce/<decision_id>.examples.jsonl` prints one chunk's `examples` per line), the last kept structure by its registered hash, and an Idempotency-Key per call ([Add labeled data](tasks.md#add-labeled-data)).

How failures come back:

- A 400 refuses the whole call before any engine call. Fix or leave out the example it names.
- Otherwise failures are per example: the rest are stored. Read each failure's `error.retryable`:
  - **`true`** (an outage, or not started in time): send it again in a new call with a new key.
  - **`false`**: sending it again fails the same way. The cause may be the content (the engine refused or couldn't read it) or the engine (an answer it couldn't use, a model other than the binding's), and `error.code` and `message` often don't say which. Don't change or drop the example on your own: show the person the failed items with their `error.code`, `message` and the binding (`get_binding`), and fix, drop or reconfigure as they decide.

Read each import's `loss_report` over its calls' `run_id`s. The dataset's loss per case should match the last refine score, and the blind set's the `blind_check` score, within noise. A permanent failure still counts in its run as a mistake (`failed` in the report), even after you drop the example from the file: compare without those. A fixed example sent again, under the same `item_id` and the same version (structure, engine and binding revision), replaces its failure when you list its run beside the original. After a binding change, runs of the two versions can't be listed together (400 `RUNS_MIXED`): compare the new runs alone. A difference means the imported examples aren't what was scored: check the file and the structure hash.

Then offer to wire the decision into the app ([Wire it into the app](tasks.md#wire-it-into-the-app)), which stops for the person before changing their code. Set the brief's `stage` to `done`. Later work on this decision is the improve flow.

## Going back

A stage can send the flow back. Set `stage` to the earlier stage and re-run the stages after it, in order.

- **A wrong or missing clause found at `refine` or `blind_check`:** go back to `brief`. Then re-label the cases the change touches at `labels`, check the structure's clause map, and go back to `refine`.
- **A severity the person wants changed:** go back to `objective`. Store the new objective, then re-score the last kept version. Reports under the old `effective_hash` no longer compare.
