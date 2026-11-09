# Design a structure

How to turn a brief into questions, attributes and aggregations. The author reads this before writing a structure, and again before re-weighting or reordering one. The full reference, with every type's arithmetic and more patterns, is Moonbounce's [Aggregations](https://docs.moonbounce.ai/learn/aggregations) page.

## Facts are attributes, judgment is a question

Read each clause of the brief and sort what it depends on:

- **A fact the app already has** (an amount, a count, a number of days, a flag, a category): make it an **attribute**. Rules, `bands` and checks compare it directly (`{"name": "attributes.days_since_delivery", "gt": 30}`), with no model call: exact, free, the same every time.
- **A judgment about content** (was the item defective? does the message say enough? is this a change of mind?): make it a **question**, one judgment each, answered by the model (P(true) for a yes/no question).
- **Two facts compared** (the request comes from the account's usual country, the typed address is the email on file, the refund is more than the order total): a **feature comparison** between two attributes, never a table of value pairs and never a question. Where a test takes a constant (`is`, `gte`, `gt`, `lte`, `lt`), put the other feature: `{"name": "attributes.request_country", "is": {"name": "attributes.usual_country"}, "normalize": "case"}`. The model judges near misses unsteadily (one letter off reads as "the same" on one run and not the next), and a table of every pair costs most of the structure and still misses a value.

Never ask a question about something the app can compute. "Was the request more than 30 days after delivery?" spends an model call to get a probability about arithmetic, and it can be wrong; `days_since_delivery` as an attribute can't be.

**How.** Declare the attribute with its type and range (`{"type": "number", "minimum": 0}`, a `boolean`, a `string` with its `enum`, or a `string` without one for open text such as an address or a name) and, for a fact already in the state, `from`: its state path, as in `reads`. Evaluate fills it from each call's `state`, validated by the declaration with no coercion, so the app's call doesn't change and the dataset's lines need nothing added:

```json
"order_total_usd": { "type": "number", "minimum": 0, "from": "state.order.order_total_usd" }
```

- **A missing or null field** takes the attribute's `default`; without one the call is a 400 naming the path. Give a default when the field can be absent (and when older traffic may lack it, since replay recomputes it from each stored case's state).
- **A value of the wrong type** (a total sent as `"300"`) or out of range is a 400. A derived string attribute refuses any label outside its `enum`, so give it an `enum` only when the field's values are a closed set. Leave the `enum` out for open text (at most 200 characters; an empty value is unknown): only `is` with another feature, and `abstained`, read it.
- **A fact the app computes** that isn't in the state (a count from another system, a lookup) has no `from`: the app sends it in `attribute_values` with each call. That changes the app's call, so ask the person before relying on one.

A fact can become an attribute at any point: while writing the brief, when the author writes the structure, or in the refine loop when a question about a fact keeps getting it wrong. The brief doesn't have to have named it.

**Switching an attribute the app already sends to `from`** is the one change that isn't free. A version that derives it refuses a value for it in `attribute_values`, so every live call that still sends it would be a 400. Tell the person: the app must stop sending that attribute when it starts sending the new version's hash, in the same release. In the dataset, drop it from the lines' `attribute_values` (`jq -c 'del(.attribute_values.<name>) | if .attribute_values == {} then del(.attribute_values) else . end' moonbounce/<decision_id>.examples.jsonl > moonbounce/scratch/<decision_id>/examples.jsonl`, then `cp` it back: never redirect onto the file jq reads), and score the old version on its own copy that keeps it.

**Keep facts out of questions.** A question sees only the state paths in its `reads`, so leave out every field that is an attribute: a model that also sees the numbers may weigh them its own way, against the exact comparisons in your rules. A path a question reads must be present on every call, so an optional field is read through its parent, and the question then sees the parent's other fields too. When an optional text field shares a parent with facts (a tracking line inside `order`, beside the totals), ask whether the app can send the text under a key of its own (`state.delivery`, always present, holding an optional `tracking`), so the question reads just that.

**Comparing two facts.** Make both attributes and compare them in a rule's condition, or in a check whose one input is the comparison (`{"type": "all", "inputs": [<the test>]}`, 1 when it holds, 0 when not), which feeds any aggregation as a number. Pick the normalization per comparison:

- **None** (exact) for codes and ids the app already writes one way.
- **`case`** trims and lower-cases both sides, for country codes or names that may differ in case.
- **`email`** is `case`, then drops a `+tag` and Gmail's dots, so `Rob.Tanaka+help@gmail.com` is `robtanaka@gmail.com`; `rob.tanakaa@corp.com` still differs from `rob.tanaka@corp.com`. It widens "the same" on every domain (`rob+x@corp.com` is `rob@corp.com`), so use `case` where a `+tag` may be a different mailbox.

The test is unknown when either side is: a missing value, an empty open string, or one the normalization leaves blank. Give the typed value a `default` of `""` and catch it with a first rule on `{"name": "aggregations.<check>", "abstained": true}` that names the safe outcome. Anything beyond the named normalizations (a domain-only match, edit distance) stays in the app's code, which sends its result as an attribute.

**Keep a question when** the fact is only in free text (a tracking line in prose, a date the customer wrote), or when the app would have to judge it to compute it. A value to compare that is only in the message text (an address the customer typed) can't come from a question, which never returns text: ask the person whether the app can put the raw value in the state (`state.requester.typed_email`), derived with `from` and compared in the structure. Ask for the raw field, never a derived boolean such as `requester_is_email_on_file`: that moves the comparison into the app's code, outside the hash, where the refine loop can't see or test it.

## Questions

- **Narrow:** one judgment per question, with `criteria` saying what counts as true and false. A question that bundles two judgments can't be blamed or fixed apart.
- **From the brief, not the cases:** state the clause's criterion, not wording drawn from particular examples.
- **`reads` only what it needs:** questions that read the same fields share one model call, and a field nobody reads is never sent. Every path a question reads must be in each call's `state` (a 400 otherwise), so read a field that can be missing through its parent: `state.order`, not `state.order.tracking`.
- **Observe first:** a question no aggregation reads is answered and recorded but moves nothing, so a new one can be watched on real traffic before it counts.

## Choose the kind of question

| The judgment is | Kind | Its answer |
| --- | --- | --- |
| Yes or no, one condition | `noul` | P(true), 0 to 1 |
| One category from a fixed set (which team, which intent, which reason) | `choice`: `criteria` maps each of 2 to 255 options to its description (or null when the name says enough) | the chosen option, every option's probability, and `confidence` |
| A graded quality on an ordered rubric (how severe, how urgent, how well it meets a bar) | `score`: `criteria` lists 2 to 10 level descriptions, lowest first; a level is its position from 0 | the score (Σ level × probability, 0 to the top level), each level's probability, and `confidence` |

- **Give a choice a catch-all** (`"other": null`) unless its categories cover every case: the model always picks one option, so a case that fits none is forced into one, often with high confidence.
- **Independent conditions stay separate nouls.** "Is it a refund request?" and "Is the customer angry?" can both be true: two nouls. Categories that exclude each other ("billing", "shipping", "technical") are one choice: three nouls can all say yes, or all say no, and the rules then route by whichever comes first.
- **A choice routes with `match`:** `{"type": "match", "input": "questions.team", "cases": {"billing": "to_billing"}, "else": "to_other"}` (`else` unless the cases cover every option). An option's probability is a 0–1 value a check or cut reads with `option`: `{"name": "questions.team", "option": "billing", "gte": 0.8}`.
- **Route an unsure answer** before routing the answer, in one `rules` (a rule can't fall through to a `match`): a first rule on `confidence` (1 when one option holds all the probability, 0 when they're even), then one `is` rule per option:

  ```json
  "route": {
    "type": "rules",
    "rules": [
      { "when": { "name": "questions.team", "confidence": true, "lt": 0.5 }, "outcome": "escalate" },
      { "when": { "name": "questions.team", "is": "billing" }, "outcome": "to_billing" },
      { "when": { "name": "questions.team", "is": "shipping" }, "outcome": "to_shipping" }
    ],
    "else": "to_other"
  }
  ```
- **A score reads as its number,** 0 to the top level, like a number attribute: a check, rule or cut names its threshold (`{"name": "questions.severity", "gte": 2}`), since a score has no default one, and `weighted_mean`, `min`, `max`, `product` and `noisy_or` refuse it (`logistic` takes it as a number). Read one level's probability with `option` (`"2"`). `match`, `is` and `in` see its most likely level, as a string (`"2"`).
- **Option names** are kept exact and can't be integers (`"0"`, `"12"`); write a word. Their order doesn't count. A score's levels keep their order, which numbers them.
- **Changing kind is a new question.** A noul turned into a choice, or options added or renamed, gets a new question hash: its answers aren't reused (the model is asked again on every case) and `expected_answer` corrections on the old question don't carry. Outcome labels do.

## Choose the combination

| The policy says | Use |
| --- | --- |
| Ordered clauses, exceptions, "X wins over Y" | `rules`: the first true rule names the outcome, `else` the rest. Conditions compare attributes and questions directly, and combine with `all`, `any` and `not`. The usual choice for a written policy. |
| A value to rank or cut | `weighted_mean` (relative, positive weights; `invert` for "not"), then `bands` to cut it into outcomes. It ranks well but is not a probability, and it can't veto. |
| Any one of several reasons is enough | `noisy_or` when the reasons are independent, `max` when they overlap. An average would dilute a single strong reason. |
| All of several must hold | `product` when independent, `min` when they overlap. |
| One thing vetoes the rest | `min` over the value and the inverted veto. |
| Several yes/no tests | `all` or `any` (each input takes a rule condition's test: `gte`, `gt`, `lte` or `lt`, default ≥ 0.5, `is` or `in`, `abstained`, or a comparison with another feature). They output 1 or 0, so they nest anywhere. |
| Two facts the same, or one above the other | A feature comparison, `{"name": <a>, "is": {"name": <b>}}` (with `normalize` `case` or `email`) or `gt: {"name": <b>}`, in a rule or a one-input `all`. Never a table of value pairs or a question. |
| A label straight to an outcome | `match` on a choice question, a string attribute or another outcome. |
| An unsure choice or score goes to a person | A first rule on the question's `confidence` (`lt`), naming the safe outcome. A `noul` has no `confidence`. |
| Weights fitted on labeled data | `logistic` (signed weights and a `bias`). Never guess its numbers: a guessed logistic looks more precise than it is, and fitting isn't offered yet. |

Group features that share a reason before weighing them: two independent ways to be "our failure" go in one `noisy_or`, and that group is one input to the `weighted_mean`. Otherwise a new question that says "no" on most cases drags every case down.

**Abstaining.** When a needed answer is missing, `bands` and `match` give their `abstain` outcome, and a rule with `{"name": "questions.x", "abstained": true}` catches it before an unknown stops the rules. Point both at the outcome the brief says is safe when the decision can't tell (often `escalate`). `weighted_mean` drops a missing input and averages the rest; `logistic`, `min`, `max`, `product` and `noisy_or` become unknown.

## The refund decision, worked

The refund policy's facts (days since delivery, refunds in the last year, the order total) are in `state.order` and become attributes derived from it with `from`; its judgments become two questions; its precedence becomes `rules`.

```json
{
  "attributes": {
    "days_since_delivery": { "type": "number", "minimum": 0, "from": "state.order.days_since_delivery" },
    "refunds_last_12_months": { "type": "number", "minimum": 0, "from": "state.order.refunds_last_12_months" },
    "order_total_usd": { "type": "number", "minimum": 0, "from": "state.order.order_total_usd" }
  },
  "questions": {
    "says_enough": {
      "type": "noul",
      "instructions": "Does the customer say why they want a refund clearly enough to decide the request?",
      "criteria": {
        "true": "They name a reason, including simply not wanting the item any more.",
        "false": "The message gives no reason, or contradicts itself."
      },
      "reads": ["state.message"]
    },
    "our_failure": {
      "type": "noul",
      "instructions": "Did the store fail the customer: a defect, damage on arrival, failure under normal use, a late or lost delivery, or a billing error?",
      "criteria": { "false": "The customer simply changed their mind, or damaged the item themselves." },
      "reads": ["state.message", "state.order"]
    }
  },
  "aggregations": {
    "caution": {
      "type": "any",
      "inputs": [
        { "name": "attributes.refunds_last_12_months", "gte": 2 },
        { "name": "attributes.order_total_usd", "gt": 250 }
      ]
    },
    "handling": {
      "type": "rules",
      "rules": [
        { "when": { "name": "questions.says_enough", "abstained": true }, "outcome": "escalate" },
        { "when": { "name": "questions.says_enough", "lt": 0.5 }, "outcome": "escalate" },
        { "when": { "name": "questions.our_failure", "abstained": true }, "outcome": "escalate" },
        { "when": { "all": [{ "name": "questions.our_failure", "gte": 0.5 }, { "name": "aggregations.caution", "gte": 0.5 }] }, "outcome": "escalate" },
        { "when": { "name": "questions.our_failure", "gte": 0.5 }, "outcome": "grant" },
        { "when": { "name": "attributes.days_since_delivery", "gt": 30 }, "outcome": "deny" }
      ],
      "else": "escalate"
    }
  },
  "result": "aggregations.handling"
}
```

`our_failure` reads `state.order` to get the optional tracking line, so the model also sees the order's numbers; with the tracking line under a key of its own, it would read that and the message only.

Read top down: an unclear message, or one the model couldn't answer about, escalates; our failure with a reason for caution escalates; our failure otherwise grants; a change of mind past 30 days denies; anything left escalates. Only two model questions run per request, and every number in the policy is an exact comparison. The brief's clause map in `work/notes.md` says which rule applies which clause. The policy's clause about a late delivery counting from the promised date is one the app can compute (days since the promised date) and send as `days_since_delivery`; if it can't, a question reading `state.order` (and so the tracking line, when there is one) judges it instead.

When `near_cuts` and `sweep_cut` apply to a structure like this, they apply to `bands`; a `rules` structure is tuned by its thresholds and the order of its rules, one change per candidate.
