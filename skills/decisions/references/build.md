# The build flow: a new decision

Use this as a library of stage procedures selected by the stored advisory plan and [intake adapter](intake.md). The adapter controls transitions, limits and effects; do not run this page as an independent workflow.

The person may bring a written policy, labeled data, both, or only a description of what they want decided. The brief is where the first version comes from, and it is the policy from then on: its Outcomes, Scope, Clauses and Edge cases are the rules the decision applies, whether they started from a document or from the person's answers. The dataset lives in the caller's repo, versioned by git. Scoring uploads the file and runs its lines as dry runs, which keep their answers and labels for about a day so `loss_report` can judge them; the upload lives as long, so later runs and the import name its lines (`items_from`) instead of sending them. Only the last stage, `import`, stores the dataset for good.

| Stage | Does | Leaves | Done when |
|---|---|---|---|
| `objective` | Establishes or reuses the requested outcomes and error tradeoffs; records the objective | Objective in the decision file | Relevant choices are confirmed or unresolved tradeoffs are reported |
| `brief` | Establishes or checks the selected policy clauses and actual input paths; owner probes are separate selected work | `moonbounce/<id>.brief.md`, `work/probes.jsonl` | The scoped brief is understood and its gaps or needed rulings are recorded |
| `cases` | Reuses eligible examples and prepares cases for the selected gaps, with provenance and reserved assessment groups protected | `moonbounce/<id>.examples.jsonl`, labeled | The selected evidence is usable and its coverage, label disputes and remaining gaps are reported |
| `first_version` | An author drafts and validates the needed candidate from the brief and allowed evidence; registration, settings and scoring are separate effects | Valid local candidate and clause map | It validates, and the requested clauses map to questions or rules |
| `refine` | Runs the requested supported comparisons on eligible cases; fitting targets receive independent label checks | Kept local candidates and comparison reports; registration only when selected and authorized | The requested stopping point or remaining allowance is reached, with kept/rejected changes reported |
| `red_team` | Attacks the pinned candidate on the selected failure paths, with independent labels and disclosed evidence limits | `work/red-<n>.jsonl`, dataset additions | The scoped attack report is complete; further repair, attacks or assessment follow the user's request |
| `blind_check` | Assesses the pinned candidate using preserved holdout groups or fresh independent cases, with their provenance and limitations | `moonbounce/<id>.blind.jsonl` | The assessment is reported without changing the candidate; any repair requires a revised request and fresh independent evidence afterwards |
| `import` | With the person's go-ahead, `import_examples` stores the dataset and runs the model, so previews and the improve flow can use it: `items_from` slices of its upload, the runs' slices, each line labeled (an unlabeled line is a 400 naming it) | Server copy of the dataset | Requested examples are verified stored, unresolved failures are reported and the import report is reconciled |

The current request and plan select the work, not the next row in this table. Reuse decisions the person has already made; a stage boundary is not a reason to ask for permission again. At a genuine unresolved choice, explain what is needed before continuing. For policy rulings, ask what the person would decide before showing your recommendation, then share it after they answer or ask; a recommendation shown first can hide what they actually mean. Record the selected procedure and evidence as execution context, and stop at the requested boundary. Commit only on the branch the person requested.

## Requested scope and stopping point

Establish the outcome the person wants through conversational intake. Use their materials, requested effects, evidence needs and remaining resources to select procedures. Do not offer a Quick/Full switch, choose work from a mode flag or treat the tables below as a mandatory sequence. A request for speed does not imply an unscored draft, permission to skip validation or a fixed amount of work.

For example, the person can ask: **"Build and register a quick, scored first version, and stop after the report. No tuning or app changes yet."** Record the scored endpoint and stopping instruction in the intake's `task`, set `draft_only` to No and `refine_now` to No from this request, and preserve any stated limits. This is an example of user intent, not a preset or an instruction to collect every catalog answer.

The plan selects whatever preparation and measurement that request needs. Reuse a valid brief, candidate, examples or current report where they already exist; author only when needed. Prepare a disclosed, relevant test subset within the requested resources when new measurement is needed, using real inputs where available and labeling synthetic evidence accurately. Its size follows the actual gaps and budget, not a cases-per-clause formula or a time promise. Protect reserved assessment groups. Do not claim unperformed checks passed or call synthetic evidence representative traffic performance.

A scored endpoint needs an actual `baseline` result, or verified current measurement that satisfies the request. Keep authoring and validation separate from registration/settings/scoring. Registration can be required for registered-version calls or initial custom-objective setup. Candidate dry runs use the stored objective, or balanced losses when none exists; check and disclose the actual report objective. Estimate evaluation cost and stay within existing authorization before scoring. Report the tested version, objective and model, dataset provenance and coverage, results, cost and remaining limitations. If scoring is blocked, report what remains unmeasured rather than presenting an untested draft as a scored result.

Stop at the requested boundary. Additional refinement, attacks, independent assessment, import or app integration remain separate user choices even when the planner recommends them. Show unresolved evidence goals as deferred work, not completed checks. The user may request a different composition or further work at any time; resume from current artifacts, receipts and remaining allowances without a mode conversion, mandatory import or fixed label-count gate.

## Why a team

One agent that writes the cases, labels them, writes the decision and scores it will pass its own test: it imagines a world, then fits it. Split the jobs across fresh subagents instead, so the decision is tested against cases its author never shaped: the author writes the first version alongside the case writers, from the brief and the person's own labels, and never sees the writers' cases. The case writers label their own cases, and those labels are checked where a wrong one would do harm: before the decision is changed to fit them ([Check before you fit](refine.md#check-before-you-fit)). The barriers are instructions and paths, not access control. They hold only if every role is a fresh subagent briefed with the paths it may read. A fork inherits your context, so no role runs as a fork.

| Role | Job | Reads | Writes |
| --- | --- | --- | --- |
| **You** (orchestrator) | Talk with the person, write the objective and brief, route paths, compare labels, take rulings, score versions, keep the front matter | Everything | The decision file's `objective`, the brief, the labeled JSONL, `work/rulings.jsonl` |
| **Case writer** | Writes the probes at `brief`, with no label; at `cases`, one of several run at once, it writes and labels the cases for its share of the clauses | The brief, the person's data with its labels, the labeled probes; at `brief`, every earlier probe | `work/probes.jsonl`; at `cases`, its own `cases-<w>.jsonl` in `moonbounce/scratch/<decision_id>/`, which you merge |
| **Labeler** | Labels cases blind from the brief alone: the ones a `refine` round is about to fix, each red-team round and the blind set | The brief, one file with no labels | `work/labels-<round>.jsonl` |
| **Author** | When selected, drafts and validates the needed candidate; authoring and selected case preparation may overlap; at `refine`, a fresh author per round edits a copy of the last kept version | The brief and its own `work/notes.md`; for the first version, the person's labeled lines (their data and the answered probes), never the case writers' cases; at `refine`, the round's report summary | The decision file's `structure` (at `first_version` only), candidate files in `moonbounce/scratch/<decision_id>/`, `work/notes.md` |
| **Red team** | When selected, writes attacks on the pinned candidate, allocating the chosen allowance across relevant clauses and writers | The brief, the decision file, `examples.jsonl` | Its own case and intended files in `moonbounce/scratch/<id>/`, merged into `work/red-<n>.jsonl` and `work/red-<n>.intended.jsonl` |
| **Blind writer** | Writes the blind set from the frozen brief | The brief and [Hard cases first](cases.md#hard-cases-first) only | `blind.jsonl` (unlabeled), `work/blind-intended.jsonl` |

**Models.** Run the case writer, the labeler, the author and the red team on the strongest model you have: their judgment is the flow's quality, and every label the labeler gets wrong becomes a question for the person. A scoring subagent, and one that maps the person's data onto the Inputs, do mechanical work: give them a smaller, cheaper model (in Claude Code, a subagent's `model`).

`work/` is `moonbounce/work/<decision_id>/`. It stays in the repo with the rest, so the flow resumes from the files alone. Never give the author the blind set, and never give the blind writer the decision file or `examples.jsonl`.

## Files

[A decision's files](files.md) describes local candidates, briefs, data and execution evidence. Verify the chosen structure hash and current or archived scoring reports before deciding what needs authoring or measurement; no stored live evaluations does not prove a candidate is untested. Selected procedures may also use:

- **The probes**, `work/probes.jsonl`: every probe written, as `{item_id: "probe-<round>-<k>", state, attribute_values?, metadata: {cell, group, origin: "probe"}}`, never relabeled or overwritten, so a later round can see what was already asked. A probe the person answered is copied into `examples.jsonl` with their label; one they wouldn't call stays only here, unlabeled and out of scoring, as a question still open.
- **The blind set**, `moonbounce/<decision_id>.blind.jsonl`: the same format, in its own file, labeled at `blind_check`.


## Stages

### `objective`

Create the brief with its front matter (`flow: build`, `stage: objective`) and the decision file if it doesn't exist (`{decision_id, objective}`).

First name the outcomes: agree with the person on the exact names the decision returns (`grant`, `escalate`, `deny`, say), each with a one-line meaning. They go in the brief's Outcomes section, and every label and severity uses them exactly.

Then tell the person what their answers are for, before asking. The refine loop keeps the version with the lowest **loss**: every wrong answer adds a weight, and the loss is their total. The weights are theirs to choose, and they should come from their use case: what each kind of wrong answer costs whoever lives with it. Only the ratios matter. Money is one way to think about it, as an example, not a figure to supply: if wrongly denying a refund loses a customer worth about $80, and a needless escalation costs about $4 of an agent's time, the denial should weigh about twenty times the escalation. Harm, time, risk or trust work the same way.

Then ask which wrong answer is worse, in plain words, one question per severity, in the question dialog when the host has one (up to 4 a call) ([What the person sees](../SKILL.md#what-the-person-sees)): each option names its weight, and each answer gets a line of feedback saying what was written and the reason you'll note, for the person to correct:

- **Two outcomes:** this is precision against recall. Ask: "Would you rather catch every <outcome>, even with some false alarms, or only flag <outcome> when you're sure?" Write `{"outcome": "<outcome>", "priority": "catch_all"}`, `"be_sure"`, or `"balanced"` when neither matters more.
- **Three or more:** ask once per mistake, each its own question: "When the right answer is `grant`, how bad is answering `deny`?", then the same for `escalate`, each as barely matters, a normal mistake, costly, or unacceptable. Write them as `mistakes` with severity `minor`, `normal`, `costly` or `severe`: weights of 0.5, 1, 3 and 10. The person can say "the rest are normal" at any point: a mistake not listed is normal. When they think in ratios ("about twenty times worse"), write a weight directly as `loss`, on the scale where an ordinary mistake is 1. A weight of 10 or more counts as severe, and the loop drops any candidate that adds a severe mistake on a human label, so keep that for the mistakes they would never accept. Tell them the weights you wrote.

Ask whether one outcome is the cheap place to land when unsure (often `escalate`): that is `fallback`, and sending a case there wrongly counts as minor. It only weighs mistakes; how the decision abstains or escalates is set in the decision's definition.

A loss target is optional. Report `target_met` when one is set; use it as a stopping condition only if the user explicitly requests that. Otherwise continuation follows the requested scope and remaining allowances. A target reached on tuning evidence does not establish independent readiness.

Write the answers into the decision file's `objective` ([Set the objective](tasks.md#set-the-objective)), and the why into the brief's Mistakes section. Store it only as a separate, selected and authorized live create/revise setup step; assessment and draft-only work preserve server settings.

### `brief`

Where the clauses come from depends on what the person brought:

- **A policy:** write the clauses from it, and keep it, quoted, under **Policy source**.
- **No policy, labeled data:** ask a few clarifying questions (what the decision is for, what it must never get wrong, what usually decides it), then write the clauses from the person's labels and their reasons.
- **Neither:** ask the same questions and draft a short set of clauses from the answers. The probes below fill it in.

Most people can't write rules cold, but they can call a case. So don't ask for a policy the person doesn't have: draft what you can and let their labels shape the rest.

Number the clauses, and never renumber them: add a new clause at the end, and leave a removed one's number unused, so reasons that cite a number stay right. Ask the person about every gap rather than guessing: an unclear clause here becomes a dispute in `refine`, when a version and a label disagree on a case it should settle.

When a clause depends on a fact the app already has (an amount, a count, days, a flag, a category), note it in the Inputs as an attribute derived from its state path, rather than a question the model judges ([Facts are attributes](design.md#facts-are-attributes-judgment-is-a-question)). The app sends nothing new: the attribute declares `from` and evaluate reads it from the state. It doesn't have to be settled now: the author or the refine loop can make the same change later.

When the person has labeled data already, read it, and its reasons, before you write the clauses. Each reason is the policy as someone applied it, so it should map to a clause. Act on what doesn't:

- **A reason no clause covers:** a missing clause. Draft it and ask the person.
- **A reason that contradicts a clause:** ask the person which is right, and settle it in the brief before they confirm it.

**Probes.** When selected, investigate the scoped unresolved boundaries within the remaining question allowance. Brief a fresh case writer for each round of probes, numbering the rounds from 1: "Read `<brief>`, the person's data in `<files>`, and the probes already asked in `<probes>` with their answers in `<examples>`. Write <n> new cases a careful reader of this brief could call either way: one fact either side of a line, two clauses pulling opposite ways, something the brief doesn't mention, wording that points one way while the facts point the other. Prioritize the inputs and boundaries relevant to the selected uncertainty. List the inputs each probe examines in `metadata.cell`; report unprobed inputs as gaps rather than exhausting them before stopping. Don't ask again what an earlier probe settled. Use only the inputs, types and ranges the brief lists. Don't decide their outcomes: a case you can't call is a better probe. Append each to `<probes>` as `{item_id: "probe-<round>-<k>", state, attribute_values?, metadata: {cell: <what it probes>, group, origin: "probe"}}`, leaving the lines already there unchanged. Read nothing else. Work only inside the repo. Write files with your file tool, never with a heredoc or `echo`; a `cp` or a `jq` to a file under `moonbounce/` is fine, but never onto a file the command reads. Run one plain command at a time: no `cd`, `;` or `&&` chains, loops, shell variables, `$(…)` or heredocs. Never put the person's data or item ids in a command." Set `<n>` from the selected unresolved questions and remaining shared allowance. Do not derive a mandatory probe count from the number of outcomes or repeat rounds until every input has been covered.

Put each probe to the person with no label and no recommendation, and ask how it should go: in the question dialog when the host has one, the round's probes queued in one call (up to 4 a call; the host shows them one at a time), the outcomes as options in a fixed order. After the call, say in a line per probe what it settled or changed in the brief, and the reason you'll record. Ask for the reason in text when you can't guess it, which is often for a probe: its answer usually adds or moves a rule. For each answer:

- When it says something the brief doesn't, or says otherwise, change the brief: a clause, an edge case, or the Scope.
- Append the probe to `examples.jsonl` with `claims: [{kind: "expected_outcome", value: <their label>}]`, `label_source: "human"` and their reason as `reason`.
- When they can't or won't call it, leave it unlabeled in `probes.jsonl`, and note the open question in the brief's Edge cases.

After a selected probe round, record what it established and what remains unresolved. Continue only when another round serves the requested evidence goal and fits the remaining question/time/spend allowance. Reuse confirmed statements; do not require every input to be probed or another brief-confirmation question merely because a round ended. A later requested investigation or repair can revisit unresolved boundaries. If a ruling changes the brief, invalidate only affected evidence as [Going back](#going-back) describes.

Write the Description last: one line of plain words, at most 200 characters, saying what is decided and its outcomes ("Decides whether a refund request is granted, escalated or denied."). Keep to what a reviewer needs to recognize the decision: the policy's rules, thresholds and names stay in the brief. Point it out when the person confirms the brief, and tell them it is stored in Moonbounce as written for reviewers to read, alongside the structure, objective and cases the later stages send. For selected, authorized live create/revise work, when the decision already exists and the sentence changed, store it with `set_description`. Assess and draft-only plans report a needed description change without storing it.

### `cases`

The case writers see the person's data with its labels, `label_source` and reasons, as the dataset lines you write first. Their labels are how the person actually decides, and the case writers need them to write cases consistent with that. Each writer labels its own cases. Nobody checks those labels yet: `refine` does, case by case, before a version is changed to fit one ([Check before you fit](refine.md#check-before-you-fit)).

When both authoring and case preparation are selected and their inputs are ready, the author can work alongside the case writers ([`first_version`](#first_version)). Write the cases as [Writing cases](cases.md) says, into `examples.jsonl`: the person's data and then cases for the selected gaps, within the requested scope and plan limits, merged and checked for duplicates, with conflicts taken to the person before `first_version`. On a return to `cases` after a brief change, brief writers only for the clauses that changed ([A return](cases.md#a-return)).

Case preparation is done when the selected evidence is usable, relevant duplicate/conflicting labels are resolved or excluded, and remaining coverage/label gaps are reported. Clauses and outcomes outside the selected measurement stay disclosed gaps rather than forcing a larger dataset. Unresolved fitting targets remain outside fitting and scoring until their labels are usable.

### `first_version`

This section separates authoring from first measurement. Use only the procedure selected by the request and intake adapter.

**Authoring.** Use [The author](author.md#the-author) when a candidate needs to be created. Reuse a current valid candidate otherwise. Authoring and case preparation may overlap when both are selected, while keeping construction evidence separate from reserved assessment groups and independently written tests. Incorporate relevant confirmed brief changes, validate the candidate through its structure upload and verify its clause map. Authoring alone registers nothing and changes no server settings.

**First measurement (`baseline`).** Pin the chosen candidate, actual report objective and model. `loss_report` uses the objective stored for that decision/environment, or balanced losses when none is stored; an objective in a local file does not configure it. Scoring under a confirmed custom objective requires `get_objective`/`set_objective` setup and checking the report's `objective` and `objective.effective_hash` ([Same objective](refine.md)). A new decision must have a registered version before `set_objective` succeeds, so registration may be required for the requested objective even when the app will send candidates inline. Perform that setup only within authorized create/revise work. An unregistered candidate of an existing decision can use its live upload hash and the already-stored objective. If no custom objective is stored, label the report as balanced-loss scoring; do not claim it used the confirmed weights or met a custom loss target. Assessment/investigation preserve settings and report incompatible setup instead of applying those writes. Use a disclosed relevant comparison set, reusing valid evidence where it satisfies the request. Estimate the selected evaluation's cost within the user's authorization and remaining limits. Registration for callable registered-version use is likewise a separately selected effect ([Register it](author.md#register-it)).

Score through [Score a dataset](tasks.md#score-a-dataset) and show the resulting report as [What the person sees](../SKILL.md#what-the-person-sees) describes. State the tested version, objective/model, dataset and label provenance, coverage limitations, loss and cost. Do not call an unmeasured candidate scored or treat a small synthetic test as evidence of traffic performance. A requested stop after this report ends the selected work; it does not automatically start refinement, import or integration.

Authoring is done when the candidate validates and its scoped clause map is complete. Measurement is done when the report is verified, or when blocked/unmeasured parts are explicitly reported; an incomplete measurement does not satisfy a scored endpoint.

### `refine`

Run [the refine loop](refine.md) only when refinement is requested, over the eligible comparison evidence. Reuse the verified measured candidate as the initial base; registration is a separate selected effect. Brief a fresh author for a supported change using the scoped brief, notes and report, editing a copy in `moonbounce/scratch/<decision_id>/` rather than overwriting the kept version. Score the uploaded candidate against the pinned base and objective, then apply the existing keep/drop rule.

Before fitting to synthetic targets, use the independent label checks in [Check before you fit](refine.md#check-before-you-fit). Disputed targets remain excluded. Reaching a stopping instruction or resource limit does not require resolving every pending ruling first; report unresolved work and stop. A kept local version is not necessarily registered or deployed; say which effects were actually selected and verified.

Continue only under the user's continuation instructions and remaining allowances. Report each round, loss-target status, kept/rejected changes and cost. A target is not a stopping instruction unless the user makes it one. Stop at the requested boundary; recommend useful later evidence without automatically starting attacks, assessment, import or app integration.

### `red_team`

When targeted attacks are selected, examine the pinned candidate on the chosen failure paths or revision diff. Refinement need not have run first. Report attacks as targeted evidence, and preserve the current candidate during assessment. Whether attack cases later become fitting data is a separate selected use; it consumes their independence for later assessment.

1. **Write.** Allocate the selected case allowance across writers and relevant clauses ([Splitting the round](red-team.md#splitting-the-round)), and brief each fresh: "Read the decision in `<brief>` and its current version in `<decision file>`, and the dataset in `<examples>`. Write `<count>` cases this version is likely to get wrong on clauses `<clauses>`, whose right outcome the brief settles. Aim for:
   - paraphrases of cases it gets right;
   - values just either side of a cut or threshold;
   - fields the brief says can be missing;
   - keyword traps, where the words point one way and the facts the other;
   - messages that mix two issues;
   - requests just out of scope.

   Every case must be one the app could really send, using only the inputs the brief lists, within their types and ranges. A case that mixes two clauses is yours only when the lower-numbered one is yours. Write each case to `<your cases>` as `{item_id: "red-<n>-<k>", state, attribute_values?, metadata: {cell: <attack>, group, origin: "red"}}`, numbering from `<first k>`. Write its attack, the outcome the brief requires and why to `<your intended>`. Work only inside the repo. Write files with your file tool, never with a heredoc or `echo`; a `cp` or a `jq` to a file under `moonbounce/` is fine, but never onto a file the command reads. Run one plain command at a time: no `cd`, `;` or `&&` chains, loops, shell variables, `$(…)` or heredocs. Never put the person's data or item ids in a command."

   Merge the writers' files into `work/red-<n>.jsonl` and `work/red-<n>.intended.jsonl`. Who writes it, and how another model's output is captured: [Who writes it](red-team.md#who-writes-it).
2. **Label.** Settle the round's labels as [Settling its labels](red-team.md#settling-its-labels) says. A disagreement that changes the brief goes back to `brief`, as any brief change does.
3. **Keep.** Add the round to the end of `examples.jsonl` and record it, as [Keeping it](red-team.md#keeping-it) says.
4. **Score.** Score the settled version once over the round's lines (`items_from` with `offset` the dataset's line count before them), add that run to its base runs (a report takes at most 50 runs a side: when the base already has 50, judge later versions on a fixed part that leaves room, the same for every version), and show the person the result by attack.
5. **Report and stop where requested.** Show attack results, provenance, cost and limitations for the pinned version. Recommend any justified repair or independent assessment without automatically selecting it. The user may stop, select further work or revise the request; no fixed menu, round count or import step determines the next action.

### `blind_check`

Select independent evidence for the requested assessment: use an eligible preserved real holdout or verified matching report where available. Check its membership, exposure history and artifact bindings first. New synthetic blind writing is selected only when fresh synthetic evidence is needed; it establishes brief consistency, not traffic performance. The procedure below applies to that selected preparation and measurement, within the assessment allowance.

For new blind writing, freeze the brief: record a sha256 of its text below the front matter as `frozen` (`awk 'n>=2; /^---$/{n++}' moonbounce/<decision_id>.brief.md | shasum -a 256`; the one command in the guides that pipes, so the person may be asked to approve it once). The front matter changes as stages pass; the text below it is what the blind writer reads. Before scoring the blind set, check the hash still matches: if the brief changed, the blind set no longer tests it.

Brief a fresh blind writer that has seen neither the decision file nor `examples.jsonl`: "Write cases for the decision in `<brief>`: up to `<assessment allowance>` cases for the selected evidence goal, mostly hard ones as `<cases guide>` § Hard cases first lists, with relevant plain calls as the allowance permits. Report any untested clauses or outcomes. Write each case to `<blind>` as `{item_id, state, attribute_values?, metadata: {cell, group, origin: "blind"}}`, using only the `state` keys and attribute values the brief's Inputs section lists, within their types and ranges, and its intended outcome with a one-line reason citing a clause to `<blind-intended>` as `{item_id, label, reason}`, never in the case itself. Read nothing else. Work only inside the repo. Write files with your file tool, never with a heredoc or `echo`; a `cp` or a `jq` to a file under `moonbounce/` is fine, but never onto a file the command reads. Run one plain command at a time: no `cd`, `;` or `&&` chains, loops, shell variables, `$(…)` or heredocs. Never put the person's data or item ids in a command."

Then:

1. Have a fresh labeler label them blind, on a copy without `metadata` (`jq -c 'del(.metadata)'`, as at `red_team`), and settle disagreements and unsure cases with the person, blind, as step 4 of [Check before you fit](refine.md#check-before-you-fit) asks (the case alone, then which side said what). A settlement that changes the brief unfreezes it: re-check the dataset's cases the change touches, as [Going back](#going-back) says.
2. Write the settled labels into `blind.jsonl`: `claims: [{kind: "expected_outcome", value}]`, `label_source` (`synthetic` where the blind writer and the labeler agreed, `human` for the person's ruling) and `reason`.
3. Upload `blind.jsonl` and score the last kept version on it once, a run per 200 lines, judged by one `loss_report`.

Report loss, wrong cases, provenance and limitations against the pinned candidate. Assessment leaves the candidate unchanged. Suggest further work where the evidence warrants it, but neither a favorable result nor a failure selects import, repair or app integration. If the user requests repair, revise the intake before editing and record which exposed assessment groups can no longer serve as independent evidence. A later assessment of a changed candidate needs fresh independent groups.

Never revise against the blind set, and never show it to the author.

### `import`

Import only when storage is selected, using [Importing a dataset](import.md). Choose the desired, compatible labeled sets and registered version actually authorized for that import; exclude groups still reserved for independent assessment. An old blind file is not automatically part of the request. Verify artifact identities and compare import reports with earlier measurements only when version, dataset, model and objective match.

Record completion of the selected storage work and any unresolved failures. App integration is a separate explicit request. Future work resumes from the current request, verified artifacts and remaining resources, not from a mandatory change to another flow.

## Going back

These are reconsideration triggers for selecting affected work, not automatic jumps through the guide. Verify what changed and what evidence remains valid, reconcile the current request and allowances, and replan only the needed work. No trigger authorizes new scoring, imports or changes on its own.

Record which artifacts or evidence changed, then select only the affected procedures needed for the current request. Reuse still-valid evidence; do not replay intervening stages merely because of their display order.

- **Changed policy or a wrong/missing clause:** inspect the affected brief, case coverage and clause map. For cases being reused, touched labels include reasons citing changed/removed clauses, situations described by new clauses or edge cases, and inputs read by a changed clause; when impact is uncertain, recheck. If repair is requested, use a fresh labeler to relabel those cases blind and remove their old labeler confirmations before fitting (the recheck command in [Check before you fit](refine.md#check-before-you-fit)). Preserve raw human rulings but reassess their applicability; disputed targets stay excluded. Prior rulings and comparisons whose bindings changed need verification; unresolved targets stay out of fitting. Assessment reports the issue without changing the candidate.
- **Changed error tradeoffs or a target:** inspect the objective and existing report. Apply a new stored objective only within an authorized operation, then recompute or remeasure only what the requested conclusion requires. Compare reports only under the same `objective.effective_hash`; a target-only change affects target interpretation rather than the underlying loss objective.
