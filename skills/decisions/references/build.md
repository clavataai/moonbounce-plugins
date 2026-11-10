# The build flow: a new decision

The person may bring a written policy, labeled data, both, or only a description of what they want decided. The brief is where the first version comes from, and it is the policy from then on: its Outcomes, Scope, Clauses and Edge cases are the rules the decision applies, whether they started from a document or from the person's answers. The dataset lives in the caller's repo, versioned by git. Scoring uploads the file and runs its lines as dry runs, which keep their answers and labels for about a day so `loss_report` can judge them; the upload lives as long, so later runs and the import name its lines (`items_from`) instead of sending them. Only the last stage, `import`, stores the dataset for good.

| Stage | Does | Leaves | Done when |
|---|---|---|---|
| `objective` | Names the outcomes, then asks which wrong answer is worse, in plain words; writes the objective | Objective in the decision file | **Stop:** the user confirms the outcomes, severities and fallback |
| `brief` | Writes the brief from the policy, the person's labeled data or clarifying questions, then puts probe cases to the person, twice as many a round as there are outcomes, who labels them blind; their answers change the brief | `moonbounce/<id>.brief.md`, `work/probes.jsonl` | **Stop:** the user confirms it. After each probe round that changed it, the user chooses whether to run another |
| `cases` | Case writers run at once, each with a share of the brief's clauses, and write the dataset around the user's own data and the labeled probes: about five cases a clause, mostly ones a reader of the brief could call either way, each labeled by its writer (`synthetic`) | `moonbounce/<id>.examples.jsonl`, labeled | Every clause tested on both sides of its line, every outcome present, no duplicates the `jq` checks find, and each conflict between two writers' labels, or with the user's own data, settled with the user |
| `first_version` | The author, started at `cases` alongside the case writers, writes the first version from the brief and the user's own labels, never the writers' cases; it takes any brief change from settling the cases, then it is registered and scored | First version in the decision file | It validates, and every brief clause maps to a question |
| `refine` | The refine loop over the dataset; before a change aims at particular wrong cases, a blind labeler checks their labels, and the user rules on the ones it disputes | Accepted versions, each registered as it is kept; loss reports | **Stop:** the user says it's good enough |
| `red_team` | A red team (another model when the user has one) attacks the version `refine` settled on; the round is labeled blind, settled with the user, scored once and joins the dataset | `work/red-<n>.jsonl`, dataset additions | **Stop** after each round: the user chooses `refine` (recommended when it broke a costly or severe mistake; the new version is attacked next), another round, `blind_check` (recommended when it broke nothing worse than a normal mistake, or after three rounds), or finish (`import` without the blind check) |
| `blind_check` | An agent that has seen neither the decision nor the examples writes a fresh set from the frozen brief, hard cases first, as the case writer does; scored once | `moonbounce/<id>.blind.jsonl` | **Stop:** the user accepts the result. Failures join the training set and the flow returns to `refine`, then `red_team`; the next freeze gets a fresh blind set |
| `import` | With the person's go-ahead, `import_examples` stores the dataset and runs the model, so previews and the improve flow can use it: `items_from` slices of its upload, the runs' slices, each line labeled (an unlabeled line is a 400 naming it) | Server copy of the dataset | Every example stored, except ones the person agreed to leave out (a retryable failure sent again in a new call; one with `retryable: false` fixed, dropped or taken to the person), and `loss_report` over the runs matches the last scores; the build flow ends |

At a **Stop**, show the person what the stage left and wait for their answer before moving on. Leaving a stage that ends in a Stop, and every move back, is the person's choice, even when this guide names the usual next stage: recommend it, and let them choose. A stage without a Stop (`cases`, `first_version`) moves on when its done-when holds. When the question is how a case should go or which of two clauses wins, ask what the person would decide before showing your recommendation, and show it once they answer or ask for it: a recommendation offered first tends to be accepted, which hides what they actually think. Ask through the host's question dialog when it has one ([What the person sees](../SKILL.md#what-the-person-sees)). When a stage is done, set the brief's `stage` to the next one and its `date` to today, and commit only when the brief's `commits` names a branch.

## Quick mode

A full build asks a lot of the person and takes an hour or more of agent time. Quick mode gets them to a registered, scored first version in about 25 minutes, then pauses before `red_team` with the decision ready to wire into their app and improve on real traffic. It runs the same stages, roles and files, with less in each, so hardening it later is the build carrying on.

**Choosing it.** As the first thing at `objective`, ask: a quick first version (about 25 minutes: fewer questions and cases, one measured version, then wire it in and improve it on real traffic) or the full build (an hour or more: more probes and cases, the refine loop, the red team, the blind check and the import). Recommend quick unless the person asks for the full checks, and don't ask when they already said which ("quick", "a draft to run on real traffic" or "an MVP" is quick mode). Record `mode: quick` or `mode: full` in the front matter; a brief without `mode` is a full build.

In quick mode, the stages change like this:

- **`objective`:** draft the outcome names, severities and fallback from what the person brought (where it doesn't say, a mistake is normal; the fallback is `escalate` only when that is one of the outcomes; with two outcomes, draft the `priority`), and confirm them as one question, with each drafted weight in its text, instead of one question each. Ask commits and the spending cap (a dollar figure: a quick build's scorings cost cents) in the same call.
- **`brief`:** one round of probes, with `<n>` the number of outcomes, then confirm the brief. Write each input's exact `state` path and type in the Inputs, since the author and the case writers both work from them. More probes wait for hardening.
- **`cases`:** brief the writers as [Writing cases](cases.md) says, with "about five hard cases a clause" replaced by "about three hard cases a clause (a boundary pair, and one collision or case the brief doesn't cover; ignore the guide's sizing line)". When the person has more than about 20 labeled lines of their own, put a spread of about 20 across outcomes in the dataset and copy the rest to `work/held-back.jsonl` for hardening. The author starts alongside the writers, as in a full build.
- **`first_version`:** as written, except the five-line check and the budget estimate: the first scoring follows at once and gives the cost per item.
- **`refine`:** score the first version once over the dataset and show its report. When the report shows cuts whose `sweep_cut` `cross_validation` holds, batch them into one numbers-only candidate, score it, check the synthetic labels among the cases it fixes as [Check before you fit](refine.md#check-before-you-fit) says for a cut move (scoring it again when a ruling moves a label), and keep and register it only by the [keep rule](refine.md#keep-or-drop-a-change). Put at most three disputed cases to the person: when more are disputed, drop the candidate and keep the first version, so no case waits for a ruling. Rewording a question waits for hardening or the improve flow, and so does the first round's sample of right cases. When the verdict hinges on one or two cases, keep the first version. Note the kept version's `loss_per_case` in `work/runs.json`. No case should be waiting for a ruling (`pending` in `work/rulings.jsonl`); rule on any that is first, since `import` refuses an unlabeled line. Then, instead of the usual end of the stage, **Stop**: show the kept version, what its score means (the loss on cases agents wrote and labeled from the brief, checked only where a change aimed at them; reviewed real traffic is the measure that counts), its cost per item and per thousand calls, and three choices:
  1. **Wire it in and pause** (recommended). The decision is registered, so the app can call it: wire it in, observe-first, and tell the person where real labels come from ([Wire it into the app](tasks.md#wire-it-into-the-app)). Set `stage: red_team` and keep `mode: quick`. The improve flow takes over once real traffic has about 50 labels from reviewers or owners; until then it would tune the decision to labels agents wrote.
  2. **Harden it now:** follow *Hardening a quick build*, below.
  3. **Finish here**, as at `refine`: go on to `import`, for the dataset only, so previews and the improve flow can use it. Say what is skipped, and how many of the person's labeled lines stay out in `work/held-back.jsonl`.

**Resuming a paused quick build.** A build at `stage: red_team` with `mode: quick` is paused: the decision is usable, and nothing is in progress. However the flow was reached, ask whether to harden it, finish it (`import`, as in choice 3) or leave it paused, before doing anything else. The improve flow can start from a paused build: it stores the dataset first when the decision has no stored cases.

**Hardening a quick build.** Set `mode: hardening` and, in the same edit, the first step's `stage`. It works as `full` with these differences, so a resumed flow knows them:

1. With `stage: brief`, run any more probe rounds you'd recommend. When there are none, start at step 2. When a round changes the brief, the kept version's clause map is checked at step 3.
2. Set `stage: cases`: a [return to `cases`](cases.md#a-return) for every clause, about two more hard cases a clause, with the person's lines in `work/held-back.jsonl` added first as [The person's data first](cases.md#the-persons-data-first) says. Each writer takes the next unused hundred of item ids and a new group prefix, as a return does. No author starts: the kept version is the first base.
3. Skip `first_version`. When the brief changed since the kept version was written, check its clause map, as [Going back](#going-back) says, and have the first refine round's author cover what's missing. Set `mode: full` and `stage: refine`, score the kept version on the whole dataset (its runs and uploads may have expired), give the person the full build's cost estimate from that scoring and check their cap, and go on as written: `refine` (its first round's sample of right cases included), `red_team`, `blind_check`, `import`. When the decision is already wired in and in use, keep its model, as [The first version](author.md) says.

## Why a team

One agent that writes the cases, labels them, writes the decision and scores it will pass its own test: it imagines a world, then fits it. Split the jobs across fresh subagents instead, so the decision is tested against cases its author never shaped: the author writes the first version alongside the case writers, from the brief and the person's own labels, and never sees the writers' cases. The case writers label their own cases, and those labels are checked where a wrong one would do harm: before the decision is changed to fit them ([Check before you fit](refine.md#check-before-you-fit)). The barriers are instructions and paths, not access control. They hold only if every role is a fresh subagent briefed with the paths it may read. A fork inherits your context, so no role runs as a fork.

| Role | Job | Reads | Writes |
| --- | --- | --- | --- |
| **You** (orchestrator) | Talk with the person, write the objective and brief, route paths, compare labels, take rulings, score versions, keep the front matter | Everything | The decision file's `objective`, the brief, the labeled JSONL, `work/rulings.jsonl` |
| **Case writer** | Writes the probes at `brief`, with no label; at `cases`, one of several run at once, it writes and labels the cases for its share of the clauses | The brief, the person's data with its labels, the labeled probes; at `brief`, every earlier probe | `work/probes.jsonl`; at `cases`, its own `cases-<w>.jsonl` in `moonbounce/scratch/<decision_id>/`, which you merge |
| **Labeler** | Labels cases blind from the brief alone: the ones a `refine` round is about to fix, each red-team round and the blind set | The brief, one file with no labels | `work/labels-<round>.jsonl` |
| **Author** | Writes the first version, starting at `cases` alongside the case writers; at `refine`, a fresh author per round edits a copy of the last kept version | The brief and its own `work/notes.md`; for the first version, the person's labeled lines (their data and the answered probes), never the case writers' cases; at `refine`, the round's report summary | The decision file's `structure` (at `first_version` only), candidate files in `moonbounce/scratch/<decision_id>/`, `work/notes.md` |
| **Red team** | Writes cases meant to break the version `refine` settled on, at `red_team`: two or three writers at once, each on a share of the clauses | The brief, the decision file, `examples.jsonl` | Its own case and intended files in `moonbounce/scratch/<id>/`, merged into `work/red-<n>.jsonl` and `work/red-<n>.intended.jsonl` |
| **Blind writer** | Writes the blind set from the frozen brief | The brief and [Hard cases first](cases.md#hard-cases-first) only | `blind.jsonl` (unlabeled), `work/blind-intended.jsonl` |

**Models.** Run the case writer, the labeler, the author and the red team on the strongest model you have: their judgment is the flow's quality, and every label the labeler gets wrong becomes a question for the person. A scoring subagent, and one that maps the person's data onto the Inputs, do mechanical work: give them a smaller, cheaper model (in Claude Code, a subagent's `model`).

`work/` is `moonbounce/work/<decision_id>/`. It stays in the repo with the rest, so the flow resumes from the files alone. Never give the author the blind set, and never give the blind writer the decision file or `examples.jsonl`.

## Files

[A decision's files](files.md) has the decision file, the brief, the dataset (`examples.jsonl`), `work/rulings.jsonl` and `work/runs.json`. A decision file already in the repo with no stored evaluations is a draft: the author starts from it at `cases`. The build flow also keeps:

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

You can ask whether they want a target; it is optional, and soft. The refine loop runs until the person says the decision is good enough, and a target never stops it on its own: reports say whether it is met (`target_met`), as a reference point. Loss per case counts each outcome equally, however many cases it has, so a target reads per outcome: `{"max_loss_per_case": 0.1}` is about one normal mistake in every ten cases of each outcome, on average across outcomes, and `0.05` one in twenty. A severe mistake counts as ten normal ones on the same outcome. If they want one, ask how often a normal mistake is acceptable for a typical outcome, write the matching target, and say what you chose and what it means.

Write the answers into the decision file's `objective` ([Set the objective](tasks.md#set-the-objective)), and the why into the brief's Mistakes section. You store the objective on the server at `first_version`, once the decision exists.

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

**Probes.** Then find the brief's blind spots with the person. Brief a fresh case writer for each round of probes, numbering the rounds from 1: "Read `<brief>`, the person's data in `<files>`, and the probes already asked in `<probes>` with their answers in `<examples>`. Write <n> new cases a careful reader of this brief could call either way: one fact either side of a line, two clauses pulling opposite ways, something the brief doesn't mention, wording that points one way while the facts point the other. Cover the inputs: every input in the brief's Inputs that could change the outcome, including each one the brief says doesn't count, gets at least one probe across the rounds, so list in `metadata.cell` the inputs each probe turns on, and favour inputs no earlier probe has turned on. Don't ask again what an earlier probe settled. Use only the inputs, types and ranges the brief lists. Don't decide their outcomes: a case you can't call is a better probe. Append each to `<probes>` as `{item_id: "probe-<round>-<k>", state, attribute_values?, metadata: {cell: <what it probes>, group, origin: "probe"}}`, leaving the lines already there unchanged. Read nothing else. Work only inside the repo. Write files with your file tool, never with a heredoc or `echo`; a `cp` or a `jq` to a file under `moonbounce/` is fine, but never onto a file the command reads. Run one plain command at a time: no `cd`, `;` or `&&` chains, loops, shell variables, `$(…)` or heredocs. Never put the person's data or item ids in a command." Set `<n>` to twice the number of outcomes (six for `grant`, `escalate` and `deny`): rounds repeat until the brief stops changing, so a short round the person answers in a couple of dialogs beats a long one.

Put each probe to the person with no label and no recommendation, and ask how it should go: in the question dialog when the host has one, the round's probes queued in one call (up to 4 a call; the host shows them one at a time), the outcomes as options in a fixed order. After the call, say in a line per probe what it settled or changed in the brief, and the reason you'll record. Ask for the reason in text when you can't guess it, which is often for a probe: its answer usually adds or moves a rule. For each answer:

- When it says something the brief doesn't, or says otherwise, change the brief: a clause, an edge case, or the Scope.
- Append the probe to `examples.jsonl` with `claims: [{kind: "expected_outcome", value: <their label>}]`, `label_source: "human"` and their reason as `reason`.
- When they can't or won't call it, leave it unlabeled in `probes.jsonl`, and note the open question in the brief's Edge cases.

What comes after a round depends on what it changed. When its answers changed the brief, ask the person whether to run another round before writing it, in a dialog: say in a line what the round changed and what is still open, and recommend another round while rounds keep changing the brief or inputs that could change the outcome are still unprobed. When a round changed nothing, write the next round aimed at the inputs still unprobed, if any, and move on when none are left. A round that only confirmed lines already settled isn't a reason to stop while inputs remain unprobed: a clause saying an input doesn't count is the kind of line the person is most likely to correct. When you move on, or the person says enough, ask them to confirm the brief. More probes can come later: when a refine round turns up a mistake the brief doesn't settle, put a few probes about it to the person before changing the decision. An answer that changes the brief is a brief change, handled as [Going back](#going-back) says.

Write the Description last: one line of plain words, at most 200 characters, saying what is decided and its outcomes ("Decides whether a refund request is granted, escalated or denied."). Keep to what a reviewer needs to recognize the decision: the policy's rules, thresholds and names stay in the brief. Point it out when the person confirms the brief, and tell them it is stored in Moonbounce as written for reviewers to read, alongside the structure, objective and cases the later stages send. When the decision already exists (a hardening run, say) and the sentence changed, store it now with `set_description`.

### `cases`

The case writers see the person's data with its labels, `label_source` and reasons, as the dataset lines you write first. Their labels are how the person actually decides, and the case writers need them to write cases consistent with that. Each writer labels its own cases. Nobody checks those labels yet: `refine` does, case by case, before a version is changed to fit one ([Check before you fit](refine.md#check-before-you-fit)).

Once the person's data is in, start the author on the first version, at the same time as the case writers ([`first_version`](#first_version)). Write the cases as [Writing cases](cases.md) says, into `examples.jsonl`: the person's data first, then the case writers, about five cases a clause, merged and checked for duplicates, with conflicts taken to the person before `first_version`. On a return to `cases` after a brief change, brief writers only for the clauses that changed ([A return](cases.md#a-return)).

Done when every clause is tested on both sides of its line, every outcome is present, every group the duplicate checks print has been read and settled, and every line is labeled, except the ones `refine` holds for a ruling (`pending` in `work/rulings.jsonl`), which wait there rather than going to the person as conflicts. Near-identical items join one group on import, so a duplicate that slips through adds nothing there; it costs one extra line in each scoring.

### `first_version`

The author started at `cases`, as [The first version](author.md) says, with the person's labeled lines as its dataset: pick them out of `examples.jsonl` before the writers' cases are merged in (`jq -c 'select(.metadata.origin == "user" or .metadata.origin == "probe")' moonbounce/<decision_id>.examples.jsonl > moonbounce/scratch/<decision_id>/author-examples.jsonl`), and leave the clause out when there are none. It writes from the brief while the cases are written, so it can't fit their labels, which nobody has checked yet. When settling the cases changed the brief, give it the changes (resume it, or brief a fresh author with its draft and the changes), and wait for a hash whose clause map matches the brief. A flow resumed at `cases` or `first_version` starts the author again when the decision file has no draft yet, and otherwise works from its draft. Register it, set its model and store the objective as that guide says. Then upload the dataset and dry-run its first five lines (`items_from` with `limit: 5`) as a check; their cost is the refine loop's budget estimate. Score the dataset from here, on the model you set. When you score the whole dataset on version 1, show the person its report as the skill's [What the person sees](../SKILL.md#what-the-person-sees) says: in a host that draws MCP Apps, the chart from one `loss_report` call with `show: true`, made by you, not a subagent. It is the base the refine loop starts from.

Done when it validates and every clause maps to a question.

### `refine`

Run [the refine loop](refine.md) over `examples.jsonl`, with the author editing and you scoring. Brief a fresh author each round with the brief, its `work/notes.md` and the round's report summary (the roll-ups, the cases you read and their reasons). It edits a copy of the last kept version in `moonbounce/scratch/<decision_id>/`, one candidate a round carrying the round's changes, and uploads it, so runs name it by hash ([Candidates](refine.md#candidates)). The decision file itself holds only the last kept version. A long-lived author re-reads every earlier round and re-emits the whole decision each time. The version registered at `first_version` is the first kept version. Don't use `create_preview` here: it replays stored cases, and the build flow has none yet.

The case writers' labels are checked here, not before: each round, before the author changes the decision for particular wrong cases, a fresh labeler labels those cases blind, and the person rules on the ones it disputes; in the first round, the person also labels a few cases the version gets right ([Check before you fit](refine.md#check-before-you-fit)).

The rounds run without asking: report each round and go on to the next ([When to stop](refine.md#when-to-stop)). The stage stops when the person says the decision is good enough, once every case waiting for a ruling (`ruling: "pending"`) has one. Report `target_met` when they set a target, but it never stops the loop on its own. Before the Stop, check the version the loop settled on is registered (each kept version is registered as it is kept), so stopping here leaves the server with it. At the Stop, offer:

1. **`red_team`** (recommended): attack the settled version before the blind check.
2. **Finish here:** check the model (`get_model`), set the brief's `stage` to `import`, and go on to [`import`](#import), whose Stop gives the import's cost before anything is stored; choosing to finish isn't the go-ahead for it. Say plainly what is skipped: no red team and no blind check, so there is no estimate of how it does on cases nobody tuned it on, and the brief isn't frozen.

### `red_team`

The dataset was written before the first version, and every version so far was tuned on it. Once the person says `refine` is good enough, a red team attacks that settled version: reasonable cases aimed at the weaknesses a reader of this version can see. It finds what the dataset doesn't cover, and its cases then join the dataset, so later versions can't reopen those holes.

1. **Write.** Split the round's 15 to 30 cases across two or three writers, each with a share of the clauses ([Splitting the round](red-team.md#splitting-the-round)), and brief each fresh: "Read the decision in `<brief>` and its current version in `<decision file>`, and the dataset in `<examples>`. Write `<count>` cases this version is likely to get wrong on clauses `<clauses>`, whose right outcome the brief settles. Aim for:
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
5. **Next: a Stop.** Show the round by attack, then recommend and let the person choose:
   - **Back to `refine`**, recommended when the round broke a mistake the objective rates costly or severe (look up the severity of its cell, right answer and outcome given, in the decision file's `objective`; the report's row `loss` is weighted by outcome, so it can read high for a normal mistake on a rare outcome). The round is now in the dataset, and the version that comes out is attacked again. As before any change aimed at particular wrong cases, check the synthetic labels it aims at ([Check before you fit](refine.md#check-before-you-fit)) and read the right cases nearest each broken one (`like`) first.
   - **Another red-team round** on the same version, while fewer than three have run.
   - **On to `blind_check`**, recommended when the round broke nothing worse than a normal mistake, or after three rounds.
   - **Finish here**, as after `refine`, saying what stays unfixed.

   The round's cases stay in the dataset whatever they choose.

### `blind_check`

First freeze the brief: record a sha256 of its text below the front matter as `frozen` (`awk 'n>=2; /^---$/{n++}' moonbounce/<decision_id>.brief.md | shasum -a 256`; the one command in the guides that pipes, so the person may be asked to approve it once). The front matter changes as stages pass; the text below it is what the blind writer reads. Before scoring the blind set, check the hash still matches: if the brief changed, the blind set no longer tests it.

Brief a fresh blind writer that has seen neither the decision file nor `examples.jsonl`: "Write cases for the decision in `<brief>`: mostly hard ones, as `<cases guide>` § Hard cases first lists, and a thin floor of plain ones, every outcome present. Write each case to `<blind>` as `{item_id, state, attribute_values?, metadata: {cell, group, origin: "blind"}}`, using only the `state` keys and attribute values the brief's Inputs section lists, within their types and ranges, and its intended outcome with a one-line reason citing a clause to `<blind-intended>` as `{item_id, label, reason}`, never in the case itself. Read nothing else. Work only inside the repo. Write files with your file tool, never with a heredoc or `echo`; a `cp` or a `jq` to a file under `moonbounce/` is fine, but never onto a file the command reads. Run one plain command at a time: no `cd`, `;` or `&&` chains, loops, shell variables, `$(…)` or heredocs. Never put the person's data or item ids in a command."

Then:

1. Have a fresh labeler label them blind, on a copy without `metadata` (`jq -c 'del(.metadata)'`, as at `red_team`), and settle disagreements and unsure cases with the person, blind, as step 4 of [Check before you fit](refine.md#check-before-you-fit) asks (the case alone, then which side said what). A settlement that changes the brief unfreezes it: re-check the dataset's cases the change touches, as [Going back](#going-back) says.
2. Write the settled labels into `blind.jsonl`: `claims: [{kind: "expected_outcome", value}]`, `label_source` (`synthetic` where the blind writer and the labeler agreed, `human` for the person's ruling) and `reason`.
3. Upload `blind.jsonl` and score the last kept version on it once, a run per 200 lines, judged by one `loss_report`.

Show the person the loss per case and the wrong cases, beside the dataset's numbers. If they accept the result, go on to `import`. If not, move the failures into `examples.jsonl`, record each in `work/rulings.jsonl` as `red_team` does (they were labeled blind already), and go back to `refine`, then `red_team` again: the version that comes out is new, so it is attacked before it is checked. The next freeze gets a fresh blind set from a fresh blind writer.

Never revise against the blind set, and never show it to the author.

### `import`

Import as [Importing a dataset](import.md) says, after its Stop: the dataset and the blind set, each with its own imports (when the person finished before `blind_check`, the dataset only: never an older blind file), the last kept version by its registered hash. Each import's loss per case should match the last refine score (or the red-team score, when the flow finished there), and the blind set's the `blind_check` score, within noise.

Then offer to wire the decision into the app ([Wire it into the app](tasks.md#wire-it-into-the-app)), which stops for the person before changing their code. Set the brief's `stage` to `done`. Later work on this decision is the improve flow.

## Going back

A stage can send the flow back. Set `stage` to the earlier stage and re-run the stages after it, in order.

- **A wrong or missing clause found at `refine` or `blind_check`:** go back to `brief`. When the change adds a clause or moves a line, have a fresh case writer add the hard cases and the boundary pair that test it at `cases`, as that stage requires for every clause, as [a return to `cases`](#cases) says. Then re-check the labels of the cases the change touches: those whose reason cites a changed or removed clause, those whose situation an added clause or edge case describes, and those whose inputs a changed clause reads (an exception that widens or narrows can move cases whose reasons cite only the general rule). When unsure whether a case is touched, re-check it. A fresh labeler labels them blind from the new brief, and the person rules on the ones it disputes ([Check before you fit](refine.md#check-before-you-fit)); remove the labeler's earlier confirmations of those cases from `work/rulings.jsonl`, since they were made under the old brief. Then check the decision's clause map, and go back to `refine`.
- **A severity the person wants changed:** go back to `objective`. Store the new objective, then re-score the last kept version. Reports under the old `effective_hash` no longer compare. A new target alone keeps it: only `target_met` changes.
