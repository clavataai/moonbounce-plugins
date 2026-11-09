# A decision's files

The files every flow keeps in the caller's repo, so a flow resumes from the repo alone. Each flow's guide says which of them it writes. `work/` is `moonbounce/work/<decision_id>/`.

- **The decision file**, `moonbounce/<decision_id>.json`: `{decision_id, structure, objective}`. A decision file already in the repo with no stored evaluations is a draft: the author starts from it.
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
- **The dataset**, `moonbounce/<decision_id>.examples.jsonl`, and the synthetic set, `moonbounce/<decision_id>.synthetic.jsonl`, in the same format. Each line is exactly one `import_examples` example: `{item_id, state, attribute_values?, metadata, claims, label_source, reason?}`.
  - `metadata` carries `cell` (what the case tests: the clause and the line, collision or silence it probes), `group` and `origin` (`user`, `probe`, `synthetic` or `red`), and on a case writer's case `clauses`: the numbers of the clauses it tests, ascending, as one string (`"3,7"`), since every `metadata` value is a string, number or boolean.
  - `group` is bookkeeping for you and the case writer: paraphrase twins with the same outcome share one, and a boundary pair whose outcomes differ (the same message at day 29 and day 31) gets two. The server doesn't read it: it groups imported cases by their content and text.
  - Every line carries `claims` and `label_source`: the person's labels on their data and the probes they answered, and the case writer's on its cases. A line waiting for the person's call has none until they give it: a writer's case that conflicts with the person's own data (`metadata.conflicts_with`), or one held for a ruling (`pending` in `work/rulings.jsonl`).
  - `label_source` is `human` only for a label the person gave: their own data, labeled by them or their team; a probe they labeled; a case they ruled on ([Check before you fit](refine.md#check-before-you-fit)). It is `synthetic` for one an agent wrote: a case writer's, including one whose label follows from a brief line the person approved without labeling the case. It is `unknown` for the person's data when nobody knows who labeled it. Only `human` labels count toward the no-new-severe-mistake rule, so an agent's label never goes in as the person's.
  - The file goes up as it is ([Upload a file](tasks.md#upload-a-file)), so runs and the import read each line as written.
- **Checks**, `work/rulings.jsonl`: each case whose label was checked in the refine loop, as `{item_id, label, labeler, version, ruling, reason, clause?}` (the case writer's label, the labeler's, the version's answer, and `ruling`: `labeler` when the labeler confirmed the label, or what the person decided), so a case is never checked twice.
- **Uploads and runs**, `work/runs.json`: each upload's `upload_id`, the file it holds (with its commit when there is one) and its `lines`; for each scoring, the version hash and the `run_id`s with the offset each read; and when each expires. Both expire after about a day, so a resumed flow uploads and re-scores what it needs.
