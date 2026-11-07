# Design a structure

How to turn a brief into questions, attributes and aggregations. The author reads this before writing a structure, and again before re-weighting or reordering one. The full reference, with every type's arithmetic and more patterns, is Moonbounce's [Aggregations](https://docs.moonbounce.ai/learn/aggregations) page.

## Facts are attributes, judgment is a question

Read each clause of the brief and sort what it depends on:

- **A fact the app already has** (an amount, a count, a number of days, a flag, a category): make it an **attribute**. Rules, `bands` and checks compare it directly (`{"name": "attributes.days_since_delivery", "gt": 30}`), with no model call: exact, free, the same every time.
- **A judgment about content** (was the item defective? does the message say enough? is this a change of mind?): make it a **question**, one judgment each, answered by the model as P(true).

Never ask a question about something the app can compute. "Was the request more than 30 days after delivery?" spends an model call to get a probability about arithmetic, and it can be wrong; `days_since_delivery` as an attribute can't be.

**How.** Declare the attribute with its type and range (`{"type": "number", "minimum": 0}`, a `boolean`, or a `string` with its `enum`) and, for a fact already in the state, `from`: its state path, as in `reads`. Evaluate fills it from each call's `state`, validated by the declaration with no coercion, so the app's call doesn't change and the dataset's lines need nothing added:

```json
"order_total_usd": { "type": "number", "minimum": 0, "from": "state.order.order_total_usd" }
```

- **A missing or null field** takes the attribute's `default`; without one the call is a 400 naming the path. Give a default when the field can be absent (and when older traffic may lack it, since replay recomputes it from each stored case's state).
- **A value of the wrong type** (a total sent as `"300"`) or out of range is a 400. A derived string attribute refuses any label outside its `enum`, so derive one only from a field whose values are a closed set.
- **A fact the app computes** that isn't in the state (a count from another system, a lookup) has no `from`: the app sends it in `attribute_values` with each call. That changes the app's call, so ask the person before relying on one.

A fact can become an attribute at any point: while writing the brief, when the author writes the structure, or in the refine loop when a question about a fact keeps getting it wrong. The brief doesn't have to have named it.

**Switching an attribute the app already sends to `from`** is the one change that isn't free. A version that derives it refuses a value for it in `attribute_values`, so every live call that still sends it would be a 400. Tell the person: the app must stop sending that attribute when it starts sending the new version's hash, in the same release. In the dataset, drop it from the lines' `attribute_values` (`jq -c 'del(.attribute_values.<name>) | if .attribute_values == {} then del(.attribute_values) else . end'`), and score the old version on its own copy that keeps it.

**Keep facts out of questions.** A question sees only the state paths in its `reads`, so leave out every field that is an attribute: a model that also sees the numbers may weigh them its own way, against the exact comparisons in your rules. A path a question reads must be present on every call, so an optional field is read through its parent, and the question then sees the parent's other fields too. When an optional text field shares a parent with facts (a tracking line inside `order`, beside the totals), ask whether the app can send the text under a key of its own (`state.delivery`, always present, holding an optional `tracking`), so the question reads just that.

**Keep a question when** the fact is only in free text (a tracking line in prose, a date the customer wrote), or when the app would have to judge it to compute it.

## Questions

- **Narrow:** one judgment per question, with `criteria` saying what counts as true and false. A question that bundles two judgments can't be blamed or fixed apart.
- **From the brief, not the cases:** state the clause's criterion, not wording drawn from particular examples.
- **`reads` only what it needs:** questions that read the same fields share one model call, and a field nobody reads is never sent. Every path a question reads must be in each call's `state` (a 400 otherwise), so read a field that can be missing through its parent: `state.order`, not `state.order.tracking`.
- **Observe first:** a question no aggregation reads is answered and recorded but moves nothing, so a new one can be watched on real traffic before it counts.

## Choose the combination

| The policy says | Use |
| --- | --- |
| Ordered clauses, exceptions, "X wins over Y" | `rules`: the first true rule names the outcome, `else` the rest. Conditions compare attributes and questions directly, and combine with `all`, `any` and `not`. The usual choice for a written policy. |
| A score to rank or cut | `weighted_mean` (relative, positive weights; `invert` for "not"), then `bands` to cut it into outcomes. It ranks well but is not a probability, and it can't veto. |
| Any one of several reasons is enough | `noisy_or` when the reasons are independent, `max` when they overlap. An average would dilute a single strong reason. |
| All of several must hold | `product` when independent, `min` when they overlap. |
| One thing vetoes the rest | `min` over the score and the inverted veto. |
| Several yes/no thresholds | `all` or `any` (each input with `gte`, `gt`, `lte` or `lt`; default ≥ 0.5). They output 1 or 0, so they nest anywhere. |
| A label straight to an outcome | `match` on a string attribute or another outcome. |
| Weights fitted on labeled data | `logistic` (signed weights and a `bias`). Never guess its numbers: a guessed logistic looks more precise than it is, and fitting isn't offered yet. |

Group features that share a reason before weighing them: two independent ways to be "our failure" go in one `noisy_or`, and that group is one input to the score. Otherwise a new question that says "no" on most cases drags every case down.

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
