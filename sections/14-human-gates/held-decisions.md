---
title: "Held decisions and the review queue"
parent: "Decision authority and escalation"
grand_parent: "Human gates, decisions and alignment"
nav_order: 1
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Held decisions and the review queue

Part of [Decision authority and escalation]({{ '/sections/14-human-gates/decision-authority.html' | relative_url }}), in [Human gates, decisions and alignment]({{ '/sections/14-human-gates/' | relative_url }}).

A held decision is a question an agent cannot answer itself, written as a document and kept until the person rules on it. A review queue is the one place where all held decisions wait. Together they move questions out of the chat, where they scroll away, into files the person reads in one sitting and the agents can find again after their context is gone.

## Why a document and not a chat message

- A chat message lives in one session's context. If the session compacts or ends, the question and its context are lost. A file survives.
- Several agents can have questions at once. A queue lets the person answer all of them in one pass instead of being interrupted by each.
- A document has room for options and reasoning. A chat question tends to be one line, which forces the person to reconstruct the context.
- The ruling is recorded where the next agent will read it.

## The parts of a held decision

![Sequence diagram. A blocked agent writes context and evidence, options with pros and cons, and a recommendation into a decision file, and adds a row with its path to the review queue. The person reads the queue, newest first, and writes a ruling in their own words into the file. The agent reads the ruling verbatim, marks the item resolved and reports that it acted on it.]({{ '/assets/images/14-human-gates/held-decision-sequence.svg' | relative_url }}){: style="width:100%"}

*A held decision, from question to ruling.*

![A held decision: the agent writes context, options with pros and cons, and a recommendation. It waits in one review queue. The person writes a ruling, and the agent records it and continues.]({{ '/assets/images/14-human-gates/held-decision.svg' | relative_url }}){: style="width:100%"}

*The same parts as a flow, from the agent's draft to the person's ruling.*

| Part | What it holds |
|---|---|
| Title and task | One line naming the decision, and the task or initiative it serves |
| Context | What the agent was doing, what it investigated, what it found |
| Options | Two or more choices, each with its real pros and cons, including "do nothing" when that is a real option |
| Recommendation | Reasoning first, then the choice, so the person can check the reasoning before seeing the answer |
| Consequences | What happens next under each option, and what it costs |
| Ruling | Left empty for the person, who writes their choice in their own words |

### A template

```markdown
## Decision: <one line>

Serves: <task or initiative>
Status: held, waiting for the person

### Context
<what the agent was doing; what it checked; the evidence>

### Options
1. <option A>. Pros: ... Cons: ...
2. <option B>. Pros: ... Cons: ...

### Recommendation
<reasoning>. Therefore option <A or B>.

### If chosen
- A: <next steps, cost, time>
- B: <next steps, cost, time>

### Ruling
<the person writes here>
```

### A filled-in example

```markdown
## Decision: index design for the orders table

Serves: speed up the order history page (p95 load is 4.2 s)
Status: held, waiting for the person

### Context
The history page filters orders by customer and sorts by date.
EXPLAIN shows a full table scan on 3.1 million rows.
Two indexes fix it in a test copy; each adds write cost.

### Options
1. Index on (created_at). Pros: also speeds the daily report. Cons: the page still
   filters 40,000 rows per customer after the index; p95 falls to 1.1 s.
2. Index on (customer_id, created_at). Pros: p95 falls to 0.08 s. Cons: does not help
   the daily report; about 3 percent slower inserts.

### Recommendation
The page is the stated goal and option 2 meets it by a wide margin; the report runs
once a night and is not slow today. Therefore option 2.

### If chosen
- 1: one migration, 10 minutes of locked writes at deploy.
- 2: one migration, 12 minutes of locked writes at deploy.

### Ruling
Option 2. We query by customer far more than by date. Deploy it on Sunday.
```

### What the agent does while a decision is held

| Situation | What the agent does |
|---|---|
| Other work in the task does not depend on the decision | continues with that work |
| Everything left depends on it | stops, says so once, and ends the run with the item in the queue |
| The decision has a deadline | puts the date in the title, so the person sees it in the index |

## The review queue

The queue is the one place where every held decision waits. Its form matters less than three properties:

1. There is exactly one, and every agent writes to it.
   - A folder of decision files, a section of the plan document, or a label on issues.
2. It starts with an index, newest first, with a path to each item.
   - The person sees the size of the job before starting.
3. The person knows to look there, because the agent says so once, at the end of a run.

```markdown
# Review queue (3 open)

| Opened | Decision | Path |
|---|---|---|
| 2026-11-09 | Approve the paid evaluation batch (est. $40) | decisions/eval-batch.md |
| 2026-11-09 | Announce the API deprecation to users | decisions/deprecation-notice.md |
| 2026-11-08 | Index design for the orders table: A or B | decisions/orders-index.md |
```

### Example: an end-of-day queue

| Item | Why it was held | The person's ruling |
|---|---|---|
| Two database index designs | architectural | "B; we query by customer more than by date" |
| Announce a deprecation to users | outward | "Yes, with a 60-day window" |
| A paid evaluation batch, estimated $40 | spending | "Approved" |

The person answers all three in ten minutes. The agents pick up the rulings the next morning and continue without asking anything else.

## After the ruling

The agent closes the loop in four steps:

1. Copy the ruling into the decision record in the person's own words.
   - A paraphrase can drift: "use option A" becomes "use A for now".
2. Mark the item resolved in the queue index.
3. Act on it, and file any new task the ruling creates, with a date.
4. Report once that it acted.

### Example: a ruling that refines an option

| | Text |
|---|---|
| Ruling | "A, but keep the old endpoint working until the end of the month." |
| Recorded | verbatim, in the decision file |
| New task | remove the old endpoint, due on the last day of the month |
| Acted on | option A implemented with the old endpoint intact |
| If paraphrased as "A" | the old endpoint would have been removed, breaking current users |

## Examples from my own harness

- Every question that survives the authority test is written as a held decision, formatted with context, options and a recommendation, and followed by a plain explanation of the trade-off.
- Each session keeps one aggregate review document listing everything I must review for its current goal, compiled from the held items and checked before it is shown to me.
- A run that cannot finish ends with one exit report naming every held item by path, instead of questions spread across the chat.

To start: one folder, `decisions/`, with the template above, and a skill or command the agent runs to open a held decision, so the format and the queue index are built into the step. A context-file line pointing at the skill is the second layer.

## Pitfalls

- A queue nobody reads is a slower way to block. Tell the person when it has items, once, at the end of a run.
- A held decision without a recommendation pushes the analysis back to the person.
- Questions that the authority test would have resolved fill the queue and hide the real ones. Run the test before writing the item.
- Two agents can hold the same question. Search the queue index before opening a new item.
- A ruling that sits unread in a file does nothing. The agent checks the queue at the start of each run.

## Sources

- Held decisions and the review queue: from the author's own harness, described conceptually.

{% include page_nav.html %}
