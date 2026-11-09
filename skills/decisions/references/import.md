# Importing a dataset

How the build flow stores a dataset for good: `import_examples` stores every line as an evaluation in this environment, with its labels, and runs the model once, so the labels become the decision's cases for previews and the improve flow. Your flow's guide says which files it imports and when.

**Stop** first: tell the person what the import does, what it will cost (about one dry run of the lines it will import), and wait for their go-ahead.

Read [Resuming an import](#resuming-an-import) when `work/runs.json` records calls of an import that didn't finish.

## The import

`import_examples` with the version by its registered hash, `items_from: {upload_id, offset}` over the file's upload, one call per 200 lines in order (offsets 0, 200, 400…), and an Idempotency-Key per call ([Add labeled data](tasks.md#add-labeled-data)). Use the file's upload: the one the last scoring read, or a new one when the file changed since or the upload expired. Each line is stored as written, `reason` and `metadata` included, so nothing is copied out by hand, which would change examples without anyone noticing. Before sending each call, record its offset and Idempotency-Key in `work/runs.json`, and the time the import began before the first; record each call's run when it returns.

How failures come back:

- A 400 refuses the whole call before any model call: stop there, fix the line it names (or, with the person, drop it), upload the file again, and go on from that call's offset over the new upload.
- Otherwise failures are per example: the rest are stored. Read each failure's `error.retryable`:
  - **`true`** (an outage, or not started in time): pick those lines by id into `moonbounce/scratch/<decision_id>/retry.jsonl` (the pick command, ids in a file jq reads first, is in the skill's [Stay inside the repo](../SKILL.md#where-a-flow-stands)), upload it, and import it in a new call with a new key.
  - **`false`**: sending it again fails the same way. The cause may be the content (the model refused or couldn't read it) or the model (an answer it couldn't use, a model other than the setting's), and `error.code` and `message` often don't say which. Don't change or drop the example on your own: show the person the failed items with their `error.code`, `message` and the model setting (`get_model`), and fix, drop or reconfigure as they decide, only once every slice has run, so no offset moves under the calls still to come.

Every line must end up stored, except ones the person agreed to drop from the file.

**Check what it stored.** When the flow scored the file first (the build flow), read each import's `loss_report` over its calls' `run_id`s, at most 50 a report, with the last scoring's runs of the same version as `base` (score the version again over the file's upload first when they have expired): the base side is then judged on exactly the imported items. With more than 50 import runs, judge them in batches of 50, each against the base runs over the same lines; lines no scoring covered have no base, so read those batches without one, and split a batch only partly covered, or score its missing base lines first. The loss per case should match the base's within noise. A permanent failure still counts in its run as a mistake (`failed` in the full report, `detail: "full"`), even after you drop the example from the file: compare without those. A fixed example sent again, under the same `item_id`, the same version and the same model, replaces its failure when you list its run beside the original. After a model change, runs of the two versions can't be listed together (400 `RUNS_MIXED`): compare the new runs alone. An import over the upload that was scored carries the same lines, so a difference beyond noise is the version hash or the model; over a new upload, check the file too.

## Resuming an import

First settle what the recorded calls left: read each recorded run back with `get_run` while it lives, and resend or take to the person any failure not yet resent or dropped, as [The import](#the-import) says. A recorded run that has expired can't be read back: check what its slice stored with `list_evaluations`, as below, and resend what isn't stored. Then go on from the first offset in `work/runs.json` with no recorded run, over the same upload, so offsets keep their lines. Resend a call that has a key but no run with that same key over the same upload: while the key lives (about a day), it replays the stored answer rather than running again. When that upload has expired, or the key has (a resend then answers a 422 or runs again), check what the call stored before sending anything: `list_evaluations` with `assignment_mode: "example"`, `since` set to when this import began (so an older example set aside under the same id isn't mistaken for this one) and each `item_id` in the slice (an import keeps a call's stored examples even when others in it fail, so check every one), and resend only what isn't stored, with a new key.
