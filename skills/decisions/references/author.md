# The first version

How the build flow gets from a brief to a registered first version: a fresh author writes the decision, you register it, set its model and store the objective. Your flow's guide says what comes after.

## The author

Brief a fresh author: "Read `<design guide>` first. Then write the decision in `<brief>` at `<decision file>`, using `<examples>`. Keep the decision file's `decision_id` and `objective` as they are. Make facts attributes and judgments questions: a fact in the state becomes an attribute that declares `from` (its state path), and in `<notes>` list each one with the clause it serves. State the brief's criteria in each question rather than wording drawn from particular cases. Read only paths the brief says the app always sends. Map every clause in the brief to the question or rule that applies it, in `<notes>`. Validate the decision file by uploading it: call `create_upload` with `{kind: "structure"}`, then `curl -sS -X PUT --data-binary @<decision file> <url>`, which answers its `structure_hash` or the issues to fix. Hand back that hash. Work only inside the repo. Write files with your file tool, never with a heredoc or `echo`; a `cp` or a `jq` to a file under `moonbounce/` is fine, but never onto a file the command reads. Run one plain command at a time: no `cd`, `;` or `&&` chains, loops, shell variables, `$(…)` or heredocs. Never put the person's data or item ids in a command." `<design guide>` is [design.md](design.md), `<examples>` the person's labeled lines when there are some: their data and the answered probes, picked out before the case writers' cases join the dataset, never those cases. Leave the clause out without any, and `<notes>` is `work/notes.md`.

An attribute derived from the state needs nothing from the person or the dataset: the app's call doesn't change, and the dataset's lines already carry the state. Add it to the brief's Inputs. Only a fact the app computes and isn't in the state (a count from another system, say) needs the person: the app would send it in `attribute_values`, so ask before relying on it.

## Register it

Register it in the environment the decision will run in, usually Production (the key's, as `whoami` says): `register_decision` with `{decision_id, structure_hash}`, the hash the author's upload answered, while the upload lives (upload the file again once it has expired). Every later run names that hash.

The first registration stores the deployment's default model as the environment's model setting, at revision 1. Then, in order:

1. **The model** (`set_model`): the one production will run (`list_models` names the choices). When the decision already has evaluations in this environment, it is in use: keep its model (`get_model`), since a change moves its live traffic at once, and change it only on the person's explicit yes.
2. **The objective** (`set_objective`), and check that `unknown_outcomes` is empty.
3. **The description**: the brief's Description sentence, as the person confirmed it ([Describe the decision](tasks.md#describe-the-decision)). A brief written without one gets one first, confirmed with the person. When `get_decision` shows a different sentence, show the person both before replacing it.

Put the version's hash in the brief's front matter. Done when it validates, is registered, and every clause maps to a question or rule.
