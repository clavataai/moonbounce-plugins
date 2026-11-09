# A red-team round

What the build and improve flows share in a red-team round: who writes it, how its labels are settled and how it joins the cases. Your flow's guide has the red team's brief, how the round is scored and what comes next.

## Who writes it

Any model can be the red team, the author's included. When the person has another model or vendor available (another agent's CLI, such as `codex exec -s read-only`), recommend using it: agents on the same model share blind spots, and another model finds weaknesses the author's can't see. Say what it sends to that vendor (the brief, the decision, and the cases or diff your flow gives it, the person's own data included), and use it only on their explicit yes. Give it the files' paths to read, never their content pasted into its quoted prompt, where a backtick or `$(…)` in the text would run in the shell. Record which model wrote each round. A read-only CLI can't write files: ask it to print one JSONL line per case, with the attack, intended label and reason under an `intended` key. Capture its output and split it into the cases file and the intended file yourself.

## Splitting the round

A writer works through its cases one after another, so one writer for a whole round is slow: 30 cases can take it over ten minutes. Run two or three writers at once, each owning a share of the clauses the round attacks (the brief's, or in the improve flow the ones the change touches) and about ten cases. Use one writer when the round is under 15 cases or attacks three clauses or fewer. A case that mixes two clauses belongs to the writer of the lower-numbered one, so no two writers write the same attack. Each writer writes its own pair of files in `moonbounce/scratch/<decision_id>/` with its own hundred of case numbers (`red-<n>-1…`, `red-<n>-101…`, `red-<n>-201…`), since parallel writes to one file can interleave. Another model's CLI splits the same way: one process a share, run at once.

Merge the writers' case files into the round's file (`jq -c . <writers' case files> > <red cases>`), and their intended files the same way, never appending to a file a command reads. Then check that no item id repeats in either: `jq -cs 'group_by(.item_id)[] | select(length > 1) | .[0].item_id' <red cases>`, and the same over `<red intended>`, print nothing when all is well.

## Settling its labels

The labeler labels the round blind, from the brief alone, on a copy without `metadata` (its attack names hint at the answer): `jq -c 'del(.metadata)' <red cases> > <unlabeled red>`. Take disagreements and unsure cases to the person, blind, as step 4 of [Check before you fit](refine.md#check-before-you-fit) asks: the case alone, then which side said what. The red team's intended labels are one side of each comparison, never the answer. A disagreement that changes the brief is a brief change, handled as your flow's guide says. Each case's label is the one both agreed on (`synthetic`) or the person's ruling (`human`).

## Keeping it

Add the round's cases, with their settled labels, to the end of the file your flow scores, and upload it again. They were labeled blind already: record each in `work/rulings.jsonl` (`labeler` where the labeler and the red team agreed, the person's ruling otherwise), so the refine loop doesn't check them again. Whatever the person chooses next, the round's cases stay.
