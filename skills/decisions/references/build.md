# Flow 1: build a decision from a policy

The person has a policy and some data, or none. The brief is where the first structure comes from. The dataset lives in the caller's repo, versioned by git, and reaches the server only at the last stage.

| Stage | Does | Leaves | Done when |
|---|---|---|---|
| `objective` | Asks which mistakes matter most, in plain words; writes the objective | Objective in the decision file | **Stop:** the user confirms the severities and fallback |
| `brief` | Writes the brief from the policy and the conversation | `moonbounce/<id>.brief.md` | **Stop:** the user confirms it |
| `cases` | A case writer fills the coverage grid, on top of the user's own data | `moonbounce/<id>.examples.jsonl` | No empty grid cell, no near-duplicates |
| `labels` | A labeler labels from the brief alone; disagreements and a sample of agreements go to the user; brief and labels are revised until they agree | Labeled JSONL, brief edits | **Stop:** the user accepts the labels |
| `structure` | The author writes the first structure from the brief | Decision file structure | It validates, and every brief clause maps to a question |
| `refine` | The refine loop over the dataset | Accepted versions, loss reports | **Stop:** good enough, or the target is met |
| `blind_check` | An agent that has seen neither the structure nor the examples writes a fresh set from the frozen brief, by the same grid; scored once | `moonbounce/<id>.blind.jsonl` | **Stop:** the user accepts the result. Failures join the training set and the flow returns to `refine`; the next freeze gets a fresh blind set |
| `import` | `import_examples` stores the dataset and runs the engines, so previews and flow 2 can use it | Server copy of the dataset | Every example stored, except ones the person agreed to leave out (a retryable failure sent again in a new call; one with `retryable: false` fixed, dropped or taken to the person), and `loss_report` over the runs matches the last scores; flow 1 ends |

At a **Stop**, show the person what the stage left and wait for their answer before moving on. When a stage is done, set the brief's `stage` to the next one and its `date` to today, then commit.

## Why a team

One agent that writes the cases, labels them, writes the structure and scores it will pass its own test: it imagines a world, then fits it. Split the jobs across fresh subagents instead, so the structure is tested against cases its author never shaped. The barriers are instructions and paths, not access control. They hold only if every role is a fresh subagent briefed with the paths it may read. A fork inherits your context, so no role runs as a fork.

| Role | Job | Reads | Writes |
| --- | --- | --- | --- |
| **You** (orchestrator) | Talk with the person, write the objective and brief, route paths, compare labels, score versions, keep the front matter | Everything | The decision file's `objective`, the brief, the labeled JSONL |
| **Case writer** | Fills the coverage grid, with intended labels kept apart | The brief, the person's data | `examples.jsonl` (unlabeled), `work/intended.jsonl` |
| **Labeler** | Labels every case from the brief alone | The brief, one unlabeled file | `work/labels.jsonl` |
| **Author** | Writes and revises the structure | The brief, the labeled `examples.jsonl`, loss reports you give it | The decision file's `structure`, `work/notes.md` |
| **Red team** | Writes cases meant to break the current kept version, at `refine` | The brief, the decision file, `examples.jsonl` | `work/red-<n>.jsonl`, `work/red-<n>.intended.jsonl` |
| **Blind writer** | Writes the blind set from the frozen brief | The brief and the grid only | `blind.jsonl` (unlabeled), `work/blind-intended.jsonl` |

`work/` is `moonbounce/work/<decision_id>/`. It is committed with the rest, so the flow resumes from the repo alone. Never give the author the blind set, and never give the blind writer the structure or `examples.jsonl`.

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
  - `group` follows the server's rule for cases: paraphrase twins with the same outcome share one, and a boundary pair whose outcomes differ (the same message at day 29 and day 31) gets two. The server merges templated near-duplicates with the same label into one case, but it can't see paraphrases, so `group` is how the dataset keeps them together.
  - Before `labels`, lines have no `claims` or `label_source`.
  - `label_source` is `human` for a label the person gave or settled, and `synthetic` for one the case writer and the labeler agreed on.
  - `mb import` uploads the file as it is.
- **The blind set**, `moonbounce/<decision_id>.blind.jsonl`: the same format, in its own file, labeled at `blind_check`.
- **Run ids**, `work/runs.json`: for each scoring, the structure hash, the file and its commit, the `run_id`s and when they expire. Runs expire after about a day, so a resumed flow re-scores what it needs.

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

Ask which mistakes matter most, in plain words: "Which is worse: granting a refund that should be denied, or denying one that should be granted?" Ask the same for every pair of outcomes the person cares about. Ask which outcome is safe when the decision can't tell (the `fallback`), and how wrong is good enough (`target`, as loss per case).

Write the answers into the decision file's `objective` ([Set the objective](tasks.md#set-the-objective)), and the why into the brief's Mistakes section. You store the objective on the server at `structure`, once the decision exists.

### `brief`

Write the brief from the policy and the conversation. Number the clauses. Ask the person about every gap rather than guessing: an unclear clause here becomes a disagreement at `labels`.

When the person has labeled data already, read it, and its reasons, before you write the clauses. Each reason is the policy as someone applied it, so it should map to a clause. Act on what doesn't:

- **A reason no clause covers:** a missing clause. Draft it and ask the person.
- **A reason that contradicts a clause:** ask the person which is right, and settle it in the brief before they confirm it.

### `cases`

Brief a fresh case writer: "Write cases for the decision in `<brief>`. Use the person's data in `<files>` first, with `metadata.origin: "user"`, turned into the `state` the brief says the app sends (a ticket's `text` may be the app's `state.message`, for instance); mark everything else `synthetic`. Fill every cell of the coverage grid with at least two cases, with paraphrase twins that have the same outcome sharing `metadata.group`, and a boundary pair (one fact apart, different outcomes) in two groups. Use only the `state` keys and attribute values the brief says the app sends. Write each case to `<examples>` as `{item_id, state, attribute_values?, metadata: {cell, group, origin}}`. Write its intended outcome and a one-line reason citing a clause to `<intended>` as `{item_id, label, reason}`, never in the case itself. Read nothing else."

When the person's data carries labels, those lines go into the dataset with their label, `label_source` and `reason` as they are. Use `label_source: "human"` only when the person says they or their team labeled it, and `unknown` when nobody knows who did. Their label is the intended one, so the case writer writes no intended label for them. The labeler still labels them blind.

Done when no grid cell is empty and no two cases are near-duplicates. Near-identical items join one group on import, so a duplicate adds nothing.

### `labels`

Give the labeler a copy without labels, so it labels every line blind, the person's labeled lines included: `jq -c 'del(.claims, .label_source, .reason, .unsure)' moonbounce/<decision_id>.examples.jsonl > moonbounce/work/<decision_id>/unlabeled.jsonl`.

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

Brief a fresh author: "Write the structure for the decision in `<brief>` at `<decision file>`, using `<examples>`. State the brief's criteria in each question rather than wording drawn from particular cases. Read only paths the brief says the app always sends. Map every clause in the brief to the question or rule that applies it, in `<notes>`. Validate with `hash_structure`."

Then register it (`register_structure`) in the environment the decision will run in. The first registration creates the decision and its binding from the deployment's default. Then do these in order:

1. Set the binding (`set_binding`) to the engine and model production will run.
2. Store the objective (`set_objective`), and check that `unknown_outcomes` is empty.
3. Run one dry-run `evaluate` on one case.

Put the structure's hash in the brief's front matter.

Done when it validates and every clause maps to a question.

### `refine`

Run [the refine loop](refine.md) over `examples.jsonl`, with the author editing and you scoring. The version registered at `structure` is the first kept version. Don't use `preview` here: it replays stored cases, and flow 1 has none yet.

**Attack rounds.** The dataset was written before the structure. A red team writes cases against the structure as it is: reasonable cases, aimed at the weaknesses a reader of this structure can see. After each kept version, run one round.

1. **Write.** Brief a fresh red team: "Read the decision in `<brief>` and its current structure in `<decision file>`, and the dataset in `<examples>`. Write 15 to 30 cases this structure is likely to get wrong, whose right outcome the brief settles. Aim for:
   - paraphrases of cases it gets right;
   - values just either side of a cut;
   - fields the brief says can be missing;
   - keyword traps, where the words point one way and the facts the other;
   - messages that mix two issues;
   - requests just out of scope.

   Every case must be one the app could really send, using only the inputs the brief lists. Write each case to `<red cases>` as `{item_id: "red-<n>-<k>", state, attribute_values?, metadata: {cell: <attack>, group, origin: "red"}}`. Write its attack, the outcome the brief requires and why to `<red intended>`."

   Any model can be the red team, the author's included. When the person has another model or vendor available (another agent's CLI, such as `codex exec -s read-only`), recommend using it for the red team. Agents on the same model share blind spots, and another model finds weaknesses the author's can't see. Record which model wrote each round. A read-only CLI can't write files: ask it to print one JSONL line per case, with the attack, intended label and reason under an `intended` key. Capture its output and split it into the two files yourself.
2. **Label.** The labeler labels the round blind, from the brief alone (the cases file has no labels). Take disagreements and unsure cases to the person, as at `labels`. The red team's intended labels are one side of each comparison, never the answer.
3. **Score.** Score the kept version on the round once, and show the person the result by attack. The round was written to break this version, so report it rather than gate on it.
4. **Keep.** Add the round's cases, with their settled labels, to `examples.jsonl`, so every later candidate is gated on them. Add the run from step 3 to the kept version's base runs, or re-score it on the grown dataset, so the base covers the new cases.

The stage stops for the person when the decision is good enough or the target is met. Also stop and ask them when a round written against a kept version gets nothing wrong, or after three rounds.

### `blind_check`

First freeze the brief: commit it, and record the commit in the front matter (`frozen: <sha>`).

Brief a fresh blind writer that has seen neither the structure nor `examples.jsonl`: "Write cases for the decision in `<brief>`, filling every cell of this grid: `<grid>`. Write cases to `<blind>` and intended labels to `<blind-intended>`, as the case writer does. Read nothing else."

Then:

1. Have a fresh labeler label them blind, and settle disagreements with the person as at `labels`. A settlement that changes the brief unfreezes it: go back to `labels` for the dataset.
2. Write the settled labels into `blind.jsonl` as at `labels`: `claims`, `label_source` and `reason`.
3. Score the last kept version on the blind set once, with one dry run and one `loss_report`.

Show the person the loss per case and the wrong cases, beside the dataset's numbers. If they accept the result, go on to `import`. If not, move the failures into `examples.jsonl` and go back to `refine`. The next freeze gets a fresh blind set from a fresh blind writer.

Never revise against the blind set, and never show it to the author.

### `import`

Store the dataset and the blind set, each with its own import: `mb import moonbounce/<decision_id>.examples.jsonl --decision moonbounce/<decision_id>.json`, then the same for the blind file. Always pass `--decision`: without it, `mb` refuses a repo with more than one decision file. `mb` reads `MOONBOUNCE_API_URL` and `MOONBOUNCE_API_KEY` from the environment, and sends the decision file's structure inline. Without `mb`, call `import_examples` with up to 200 examples a call, the last kept structure inline or by its registered hash, and an Idempotency-Key per call ([Add labeled data](tasks.md#add-labeled-data)).

How failures come back:

- A 400 refuses the whole call before any engine call. Fix or leave out the example it names.
- Otherwise failures are per example: the rest are stored. Read each failure's `error.retryable`:
  - **`true`** (an outage, or not started in time): send it again in a new call with a new key.
  - **`false`**: sending it again fails the same way. The cause may be the content (the engine refused or couldn't read it) or the engine (an answer it couldn't use, a model other than the binding's), and `error.code` and `message` often don't say which. Don't change or drop the example on your own: show the person the failed items with their `error.code`, `message` and the binding (`get_binding`), and fix, drop or reconfigure as they decide.

Read each import's `loss_report` over its calls' `run_id`s (`mb import` prints it). The dataset's loss per case should match the last refine score, and the blind set's the `blind_check` score, within noise. A permanent failure still counts in its run as a mistake (`failed` in the report), even after you drop the example from the file: compare without those. A fixed example sent again, under the same `item_id` and the same version (structure, engine and binding revision), replaces its failure when you list its run beside the original. After a binding change, runs of the two versions can't be listed together (400 `RUNS_MIXED`): compare the new runs alone. A difference means the imported examples aren't what was scored: check the file and the structure hash.

Then put the decision file in the caller's code, so the app sends that structure. Set the brief's `stage` to `done`. Later work on this decision is flow 2.

## Going back

A stage can send the flow back. Set `stage` to the earlier stage and re-run the stages after it, in order.

- **A wrong or missing clause found at `refine` or `blind_check`:** go back to `brief`. Then re-label the cases the change touches at `labels`, check the structure's clause map, and go back to `refine`.
- **A severity the person wants changed:** go back to `objective`. Store the new objective, then re-score the last kept version. Reports under the old `effective_hash` no longer compare.
