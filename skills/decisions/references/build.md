# The build flow: a new decision

The person may bring a written policy, labeled data, both, or only a description of what they want decided. The brief is where the first structure comes from, and it is the policy from then on: its Outcomes, Scope, Clauses and Edge cases are the rules the decision applies, whether they started from a document or from the person's answers. The dataset lives in the caller's repo, versioned by git. Scoring sends its items to Moonbounce as dry runs, which keep the run (the items as sent, answers, labels and attribute values) for about a day so `loss_report` can judge it and a later run can name its items (`items_from`); only the last stage, `import`, stores the dataset for good.

| Stage | Does | Leaves | Done when |
|---|---|---|---|
| `objective` | Names the outcomes, then asks which wrong answer is worse, in plain words; writes the objective | Objective in the decision file | **Stop:** the user confirms the outcomes, severities and fallback |
| `brief` | Writes the brief from the policy, the person's labeled data or clarifying questions, then puts 5 to 10 probe cases to the person, who labels them blind; their answers change the brief | `moonbounce/<id>.brief.md`, `work/probes.jsonl` | **Stop:** the user confirms it, once new probes stop changing it |
| `cases` | A case writer writes the dataset around the user's own data and the labeled probes: cases a reader of the brief could call either way first, then a thin floor of plain ones | `moonbounce/<id>.examples.jsonl` | Every clause tested on both sides of its line, every outcome present, no near-duplicates |
| `labels` | A labeler labels from the brief alone; disagreements, unsure cases and a sample of agreements go to the user, who labels the sample blind; brief and labels are revised until they agree | Labeled JSONL, brief edits | **Stop:** the user accepts the labels |
| `structure` | The author writes the first structure from the brief | Decision file structure | It validates, and every brief clause maps to a question |
| `refine` | The refine loop over the dataset | Accepted versions, loss reports | **Stop:** the user says it's good enough |
| `red_team` | A red team (another model when the user has one) attacks the version `refine` settled on; the round is labeled blind, settled with the user, scored once and joins the dataset | `work/red-<n>.jsonl`, dataset additions | **Stop:** a round breaks nothing worse than a normal mistake, or after three rounds. A round that breaks a costly or severe one goes back to `refine` first, then attacks the new version |
| `blind_check` | An agent that has seen neither the structure nor the examples writes a fresh set from the frozen brief, hard cases first, as the case writer does; scored once | `moonbounce/<id>.blind.jsonl` | **Stop:** the user accepts the result. Failures join the training set and the flow returns to `refine`, then `red_team`; the next freeze gets a fresh blind set |
| `import` | With the person's go-ahead, `import_examples` stores the dataset and runs the model, so previews and the improve flow can use it | Server copy of the dataset | Every example stored, except ones the person agreed to leave out (a retryable failure sent again in a new call; one with `retryable: false` fixed, dropped or taken to the person), and `loss_report` over the runs matches the last scores; the build flow ends |

At a **Stop**, show the person what the stage left and wait for their answer before moving on. When the question is how a case should go or which of two clauses wins, ask what the person would decide before showing your recommendation, and show it once they answer or ask for it: a recommendation offered first tends to be accepted, which hides what they actually think. When a stage is done, set the brief's `stage` to the next one and its `date` to today, and commit only when the brief's `commits` names a branch.

## Why a team

One agent that writes the cases, labels them, writes the structure and scores it will pass its own test: it imagines a world, then fits it. Split the jobs across fresh subagents instead, so the structure is tested against cases its author never shaped. The barriers are instructions and paths, not access control. They hold only if every role is a fresh subagent briefed with the paths it may read. A fork inherits your context, so no role runs as a fork.

| Role | Job | Reads | Writes |
| --- | --- | --- | --- |
| **You** (orchestrator) | Talk with the person, write the objective and brief, route paths, compare labels, score versions, keep the front matter | Everything | The decision file's `objective`, the brief, the labeled JSONL |
| **Case writer** | Writes the probes at `brief`, with no label; then the dataset around the person's data and the probes, keeping its own intended labels out of the cases | The brief, the person's data with its labels, the labeled probes; at `brief`, every earlier probe | `work/probes.jsonl`, `examples.jsonl` (its cases unlabeled, appended), `work/intended.jsonl` |
| **Labeler** | Labels every case from the brief alone | The brief, one unlabeled file | `work/labels.jsonl` |
| **Author** | Writes the structure, then edits it each round, a fresh author per round | The brief, the labeled `examples.jsonl`, its own `work/notes.md`, the round's report summary | The decision file's `structure`, `work/notes.md` |
| **Red team** | Writes cases meant to break the version `refine` settled on, at `red_team` | The brief, the decision file, `examples.jsonl` | `work/red-<n>.jsonl`, `work/red-<n>.intended.jsonl` |
| **Blind writer** | Writes the blind set from the frozen brief | The brief and [Hard cases first](#hard-cases-first) only | `blind.jsonl` (unlabeled), `work/blind-intended.jsonl` |

**Models.** Run the case writer, the author and the red team on the strongest model you have: their judgment is the flow's quality. Re-checks after a brief edit, and a scoring subagent when you use one, are mechanical: give them a smaller, cheaper model (in Claude Code, a subagent's `model`).

`work/` is `moonbounce/work/<decision_id>/`. It stays in the repo with the rest, so the flow resumes from the files alone. Never give the author the blind set, and never give the blind writer the structure or `examples.jsonl`.

## Files

- **The decision file**, `moonbounce/<decision_id>.json`: `{decision_id, structure, objective}`. A decision file already in the repo with no stored evaluations is a draft: the author starts from it at `structure`.
- **The brief**, `moonbounce/<decision_id>.brief.md`. It holds the flow's front matter (see the skill's [Where a flow stands](../SKILL.md#where-a-flow-stands)) and these sections:
  - First the policy, in sections the person can hand to their team as it is:
    - **Outcomes**: each outcome with a one-line definition, and what `escalate` or an abstention means.
    - **Scope**: what is in scope and what is out.
    - **Clauses**: a numbered list of the rules the decision applies.
    - **Edge cases**: known edge cases and how they go.
  - Then what Moonbounce needs:
    - **Mistakes**: why the objective's severities were chosen.
    - **Inputs**: the `state` keys and attribute values the app sends, each with its type and range (whole days, US dollars to the cent, one of a fixed list), and which can be missing. The red team and the case writer write only cases the app could send, so a range left out invites attacks on values that never occur.
    - **Policy source**: a supplied policy, quoted from its source, with a line saying the Clauses supersede it where they differ; or a line saying none was supplied and the Clauses were written with the person.
- **The dataset**, `moonbounce/<decision_id>.examples.jsonl`. Each line is exactly one `import_examples` example: `{item_id, state, attribute_values?, metadata, claims, label_source, reason?}`.
  - `metadata` carries `cell` (what the case tests: the clause and the line, collision or silence it probes), `group` and `origin` (`user`, `probe`, `synthetic` or `red`).
  - `group` is bookkeeping for you and the case writer: paraphrase twins with the same outcome share one, and a boundary pair whose outcomes differ (the same message at day 29 and day 31) gets two. The server doesn't read it: it groups imported cases by their content and text.
  - Before `labels`, only the person's own labels carry `claims` and `label_source`: their labeled data, and the probes they answered.
  - `label_source` is `human` only for a label the person gave: their own data, labeled by them or their team; a probe they labeled; a case whose label they chose when settling it. It is `synthetic` for one the case writer and the labeler agreed on, including a case whose label follows from a brief line the person approved without labeling the case. It is `unknown` for the person's data when nobody knows who labeled it. Only `human` labels count toward the no-new-severe-mistake rule, so an agent's label never goes in as the person's.
  - `import_examples` takes the lines as they are.
- **The probes**, `work/probes.jsonl`: every probe written, as `{item_id: "probe-<round>-<k>", state, attribute_values?, metadata: {cell, group, origin: "probe"}}`, never relabeled or overwritten, so a later round can see what was already asked. A probe the person answered is copied into `examples.jsonl` with their label; one they wouldn't call stays only here, unlabeled and out of scoring, as a question still open.
- **The blind set**, `moonbounce/<decision_id>.blind.jsonl`: the same format, in its own file, labeled at `blind_check`.
- **Run ids**, `work/runs.json`: for each scoring, the structure hash, the file scored (with its commit when there is one), the `run_id`s and when they expire. Runs expire after about a day, so a resumed flow re-scores what it needs.

## Hard cases first

The cases are there to find what the brief and the structure get wrong, so most of them should be hard. Two agents reading the same brief agree on easy cases and learn nothing from them; the cases that teach are the ones a careful reader of the brief could call either way. The case writer and the blind writer look for:

- one fact either side of a line, where it alone decides the outcome (the order at $250 and at $251, day 30 and day 31);
- two clauses pulling opposite ways on the same case;
- what the policy doesn't mention;
- wording that hedges or misleads: keywords that point one way while the facts point the other;
- the message and the order disagreeing;
- phrasing and form: indirect, multilingual, long or short, irrelevant detail, fields the brief says can be missing.

Beside them, a thin floor of plain cases, one or two per outcome and per clause, so the structure is checked on the obvious calls too. Every outcome appears, `abstain` and `escalate` included. Each case names in its intended reason the clause and the line it tests.

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

Where the clauses come from depends on what the person brought:

- **A policy:** write the clauses from it, and keep it, quoted, under **Policy source**.
- **No policy, labeled data:** ask a few clarifying questions (what the decision is for, what it must never get wrong, what usually decides it), then write the clauses from the person's labels and their reasons.
- **Neither:** ask the same questions and draft a short set of clauses from the answers. The probes below fill it in.

Most people can't write rules cold, but they can call a case. So don't ask for a policy the person doesn't have: draft what you can and let their labels shape the rest.

Number the clauses, and never renumber them: add a new clause at the end, and leave a removed one's number unused, so reasons that cite a number stay right. Ask the person about every gap rather than guessing: an unclear clause here becomes a disagreement at `labels`.

When a clause depends on a fact the app already has (an amount, a count, days, a flag, a category), note it in the Inputs as an attribute derived from its state path, rather than a question the model judges ([Facts are attributes](design.md#facts-are-attributes-judgment-is-a-question)). The app sends nothing new: the attribute declares `from` and evaluate reads it from the state. It doesn't have to be settled now: the author or the refine loop can make the same change later.

When the person has labeled data already, read it, and its reasons, before you write the clauses. Each reason is the policy as someone applied it, so it should map to a clause. Act on what doesn't:

- **A reason no clause covers:** a missing clause. Draft it and ask the person.
- **A reason that contradicts a clause:** ask the person which is right, and settle it in the brief before they confirm it.

**Probes.** Then find the brief's blind spots with the person. Brief a fresh case writer for each round of probes, numbering the rounds from 1: "Read `<brief>`, the person's data in `<files>`, and the probes already asked in `<probes>` with their answers in `<examples>`. Write 5 to 10 new cases a careful reader of this brief could call either way: one fact either side of a line, two clauses pulling opposite ways, something the brief doesn't mention, wording that points one way while the facts point the other. Don't ask again what an earlier probe settled. Use only the inputs, types and ranges the brief lists. Don't decide their outcomes: a case you can't call is a better probe. Append each to `<probes>` as `{item_id: "probe-<round>-<k>", state, attribute_values?, metadata: {cell: <what it probes>, group, origin: "probe"}}`, leaving the lines already there unchanged. Read nothing else." Write more than 10 in a round only when the brief leaves many things open.

Put each probe to the person with no label and no recommendation, and ask how it should go and why. For each answer:

- When it says something the brief doesn't, or says otherwise, change the brief: a clause, an edge case, or the Scope.
- Append the probe to `examples.jsonl` with `claims: [{kind: "expected_outcome", value: <their label>}]`, `label_source: "human"` and their reason as `reason`.
- When they can't or won't call it, leave it unlabeled in `probes.jsonl`, and note the open question in the brief's Edge cases.

When a round of answers changed the brief, write a few more probes against the new text. Stop when new probes stop changing it, or when the person says enough, then ask them to confirm the brief. More probes can come later: when a refine round turns up a mistake the brief doesn't settle, put a few probes about it to the person before changing the structure. An answer that changes the brief is a brief change, handled as [Going back](#going-back) says.

### `cases`

Give the case writer the person's data as it is, labels, `label_source` and reasons included. Their labels are how the person actually decides, and the case writer needs them to write cases consistent with that. What it keeps out of the cases is its own intended labels; the labeler is the one that never sees a label.

Brief a fresh case writer: "Write cases for the decision in `<brief>`. Use the person's data in `<files>` first, with `metadata.origin: "user"`, turned into the `state` the brief says the app sends (a ticket's `text` may be the app's `state.message`, for instance); mark everything else `synthetic`. The labeled probes in `<examples>` are already in. Write mostly hard cases, as `<build guide>` § Hard cases first lists, and a thin floor of plain ones: one or two per outcome and per clause. Put a boundary pair (one fact apart, different outcomes) in two groups, and paraphrase twins with the same outcome in one. Use only the `state` keys, attribute values, types and ranges the brief lists. Append each case to `<examples>` as `{item_id, state, attribute_values?, metadata: {cell, group, origin}}`, leaving the lines already there (the answered probes) unchanged. Write its intended outcome and a one-line reason citing a clause to `<intended>` as `{item_id, label, reason}`, never in the case itself. The person's data carries their labels: treat them as precedent. Write none for their lines, give a synthetic case close to one of their lines the same label, and where the brief would label such a case otherwise, list both item ids in `<intended>` with `label: null` and a reason naming the conflict, instead of choosing. Read nothing else."

When the person's data carries labels, those lines go into the dataset with their label, `label_source` and `reason` as they are. Use `label_source: "human"` only when the person says they or their team labeled it, and `unknown` when nobody knows who did. Their labels, and the labeled probes', are the intended ones, so the case writer writes no intended label for those lines. The labeler still labels them blind. Take each conflict the case writer listed to the person at `labels`, with the disagreements.

Done when every clause is tested on both sides of its line, every outcome is present, and no two cases are near-duplicates. Near-identical items join one group on import, so a duplicate adds nothing.

### `labels`

Give the labeler a copy without labels or `metadata` (its cell names and origins hint at the answer), so it labels every line blind, the person's labeled lines included: `jq -c 'del(.claims, .label_source, .reason, .unsure, .metadata)' moonbounce/<decision_id>.examples.jsonl > moonbounce/work/<decision_id>/unlabeled.jsonl`.

Brief a fresh labeler: "Label each case in `<unlabeled>` using only `<brief>`. For each, give the outcome, whether you are unsure, and a one-line reason citing a clause. When the brief doesn't settle a case, mark it unsure and say what the brief is missing. Write `<labels>` as `{item_id, label, unsure, reason}`."

Compare the labeler's labels with the intended ones, and take these to the person:

- every disagreement;
- every case the labeler marked unsure;
- a sample of agreements, about one in ten and at least five, which the person labels blind: show the cases without either agent's label, and compare afterwards.

Same-model writers and labelers rarely disagree outright, so the unsure cases and the person's blind sample carry most of the signal. When the person differs from the agents on more than one or two sampled agreements, the brief has a gap both agents share: settle it, and sample again until the person agrees.

When the labeler disagrees with one of the person's own labels, show the person their reason beside the labeler's. Either the brief misses what their reason says, and the brief changes, or the old label was wrong, and they relabel it. Their reason stays with the label unless they change the label.

Each disagreement is ambiguity in the brief, not noise. Settle it with the person, update the brief or drop the case, and never average two labels. When the brief changes, the case writer and the labeler re-check only the cases the change touches: those whose intended or given reason cites a changed or removed clause, those whose situation an added clause or edge case describes, and those whose inputs a changed clause reads (an exception that widens or narrows can move cases whose reasons cite only the general rule). When unsure whether a case is touched, re-check it. A re-check is mechanical, so a smaller model can do it.

Write the settled labels into `examples.jsonl`:

- `claims: [{kind: "expected_outcome", value}]`;
- `label_source`: `human` when the person gave the label (they labeled the case, or chose its label when settling it), `synthetic` when the two agents agreed, including after a brief edit the person approved;
- `reason`.

Every label must be an outcome the brief's Outcomes section names, so the structure can produce it.

### `structure`

Brief a fresh author: "Read `<design guide>` first. Then write the structure for the decision in `<brief>` at `<decision file>`, using `<examples>`. Make facts attributes and judgments questions: a fact in the state becomes an attribute that declares `from` (its state path), and in `<notes>` list each one with the clause it serves. State the brief's criteria in each question rather than wording drawn from particular cases. Read only paths the brief says the app always sends. Map every clause in the brief to the question or rule that applies it, in `<notes>`. Validate the decision file with `validate_decision` (or `mb validate <decision file> --local`)."

An attribute derived from the state needs nothing from the person or the dataset: the app's call doesn't change, and the dataset's lines already carry the state. Add it to the brief's Inputs. Only a fact the app computes and isn't in the state (a count from another system, say) needs the person: the app would send it in `attribute_values`, so ask before relying on it.

Then register it (`register_decision`, or `mb register`) in the environment the decision will run in. The first registration stores the deployment's default model as the environment's model setting, at revision 1. Then do these in order:

1. Set the model (`set_model`) to the one production will run (`list_models` names the choices). When the decision already has evaluations in this environment, it is in use: a model change moves its live traffic, so stop and ask the person first.
2. Store the objective (`set_objective`), and check that `unknown_outcomes` is empty.
3. Run one dry-run `evaluate` on one case.

Put the structure's hash in the brief's front matter.

Done when it validates and every clause maps to a question.

### `refine`

Run [the refine loop](refine.md) over `examples.jsonl`, with the author editing and you scoring. Brief a fresh author each round with the brief, its `work/notes.md` and the round's report summary (the roll-ups, the cases you read and their reasons), and have it edit the decision file in place, one change per candidate, rather than rewrite the structure. A long-lived author re-reads every earlier round and re-emits the whole structure each time. The version registered at `structure` is the first kept version. Don't use `create_preview` here: it replays stored cases, and the build flow has none yet.

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

   Every case must be one the app could really send, using only the inputs the brief lists, within their types and ranges. Write each case to `<red cases>` as `{item_id: "red-<n>-<k>", state, attribute_values?, metadata: {cell: <attack>, group, origin: "red"}}`. Write its attack, the outcome the brief requires and why to `<red intended>`."

   Any model can be the red team, the author's included. When the person has another model or vendor available (another agent's CLI, such as `codex exec -s read-only`), recommend using it: agents on the same model share blind spots, and another model finds weaknesses the author's can't see. Say that it sends the brief, the structure and the dataset (the person's own data included) to that vendor, and use it only on their explicit yes. Record which model wrote each round. A read-only CLI can't write files: ask it to print one JSONL line per case, with the attack, intended label and reason under an `intended` key. Capture its output and split it into the two files yourself.
2. **Label.** The labeler labels the round blind, from the brief alone, on a copy without `metadata` (its attack names hint at the answer): `jq -c 'del(.metadata)' <red cases> > <unlabeled red>`. Take disagreements and unsure cases to the person, as at `labels`. The red team's intended labels are one side of each comparison, never the answer. A disagreement that changes the brief goes back to `brief`, as any brief change does.
3. **Score.** Score the settled version on the round once, and show the person the result by attack.
4. **Keep.** Add the round's cases, with their settled labels, to `examples.jsonl`, and add the run from step 3 to the settled version's base runs.
5. **Next.** When the round broke a mistake the objective rates costly or severe (look up the severity of its cell, right answer and outcome given, in the decision file's `objective`; the report's row `loss` is weighted by outcome, so it can read high for a normal mistake on a rare outcome), go back to `refine` to fix it, with the round now in the dataset, and attack the version that comes out of it. As before any change aimed at particular wrong cases, read the right cases nearest each broken one (`like`) first. Stop and ask the person when a round breaks nothing worse than a normal mistake, or after three rounds; then go on to `blind_check`. Cheap breaks still join the dataset, and the person can ask for another `refine` round on them.

### `blind_check`

First freeze the brief: record a sha256 of its text below the front matter as `frozen` (`awk 'n>=2; /^---$/{n++}' moonbounce/<decision_id>.brief.md | shasum -a 256`). The front matter changes as stages pass; the text below it is what the blind writer reads. Before scoring the blind set, check the hash still matches: if the brief changed, the blind set no longer tests it.

Brief a fresh blind writer that has seen neither the structure nor `examples.jsonl`: "Write cases for the decision in `<brief>`: mostly hard ones, as `<build guide>` § Hard cases first lists, and a thin floor of plain ones, every outcome present. Write each case to `<blind>` as `{item_id, state, attribute_values?, metadata: {cell, group, origin: "blind"}}`, using only the `state` keys and attribute values the brief's Inputs section lists, within their types and ranges, and its intended outcome with a one-line reason citing a clause to `<blind-intended>` as `{item_id, label, reason}`, never in the case itself. Read nothing else."

Then:

1. Have a fresh labeler label them blind, on a copy without `metadata` (`jq -c 'del(.metadata)'`, as at `red_team`), and settle disagreements with the person as at `labels`. A settlement that changes the brief unfreezes it: go back to `labels` for the dataset.
2. Write the settled labels into `blind.jsonl` as at `labels`: `claims`, `label_source` and `reason`.
3. Score the last kept version on the blind set once, with one dry run and one `loss_report`.

Show the person the loss per case and the wrong cases, beside the dataset's numbers. If they accept the result, go on to `import`. If not, move the failures into `examples.jsonl` and go back to `refine`, then `red_team` again: the version that comes out is new, so it is attacked before it is checked. The next freeze gets a fresh blind set from a fresh blind writer.

Never revise against the blind set, and never show it to the author.

### `import`

**Stop** first: tell the person what the import does (it stores every example as an evaluation in this environment, runs the models once, and makes the labels the decision's cases for previews and the improve flow), what it will cost (about one dry run of the dataset), and wait for their go-ahead.

Store the dataset and the blind set, each with its own import: `import_examples` with up to 200 examples a call, the file's lines as they are (`jq -cs '_nwise(200)' moonbounce/<decision_id>.examples.jsonl` prints one chunk's `examples` per line), the last kept structure by its registered hash, and an Idempotency-Key per call ([Add labeled data](tasks.md#add-labeled-data)).

How failures come back:

- A 400 refuses the whole call before any model call. Fix or leave out the example it names.
- Otherwise failures are per example: the rest are stored. Read each failure's `error.retryable`:
  - **`true`** (an outage, or not started in time): send it again in a new call with a new key.
  - **`false`**: sending it again fails the same way. The cause may be the content (the model refused or couldn't read it) or the model (an answer it couldn't use, a model other than the setting's), and `error.code` and `message` often don't say which. Don't change or drop the example on your own: show the person the failed items with their `error.code`, `message` and the model setting (`get_model`), and fix, drop or reconfigure as they decide.

Read each import's `loss_report` over its calls' `run_id`s. The dataset's loss per case should match the last refine score, and the blind set's the `blind_check` score, within noise. A permanent failure still counts in its run as a mistake (`failed` in the full report, `detail: "full"`), even after you drop the example from the file: compare without those. A fixed example sent again, under the same `item_id`, the same version and the same model, replaces its failure when you list its run beside the original. After a model change, runs of the two versions can't be listed together (400 `RUNS_MIXED`): compare the new runs alone. A difference means the imported examples aren't what was scored: check the file and the structure hash.

Then offer to wire the decision into the app ([Wire it into the app](tasks.md#wire-it-into-the-app)), which stops for the person before changing their code. Set the brief's `stage` to `done`. Later work on this decision is the improve flow.

## Going back

A stage can send the flow back. Set `stage` to the earlier stage and re-run the stages after it, in order.

- **A wrong or missing clause found at `refine` or `blind_check`:** go back to `brief`. When the change adds a clause or moves a line, have the case writer add the hard cases and the boundary pair that test it at `cases`, as that stage requires for every clause. Then re-label the cases the change touches at `labels`, check the structure's clause map, and go back to `refine`.
- **A severity the person wants changed:** go back to `objective`. Store the new objective, then re-score the last kept version. Reports under the old `effective_hash` no longer compare. A new target alone keeps it: only `target_met` changes.
