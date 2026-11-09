---
title: "Held decisions and the review queue"
parent: "Decision authority and escalation"
grand_parent: "Human gates, decisions and alignment"
nav_order: 1
---

{% include page_nav.html %}

# Held decisions and the review queue

As of 2026-11-09.

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

![A held decision: the agent writes context, options with pros and cons, and a recommendation. It waits in one review queue. The person writes a ruling, and the agent records it and continues.]({{ '/assets/images/14-human-gates/held-decision.svg' | relative_url }})

| Part | What it holds |
|---|---|
| Title and task | One line naming the decision, and the task or initiative it serves |
| Context | What the agent was doing, what it investigated, what it found |
| Options | Two or more choices, each with honest pros and cons, including "do nothing" when that is real |
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

## The review queue

The queue can be a folder of decision files, a section of the plan document, or a label on issues. What matters is that there is one, that every agent writes to it, and that the person knows to look there. A useful queue file starts with a list of what is waiting, newest first, with a path to each item, so the person can see the size of the job before starting.

### Example: an end-of-day queue

At the end of a long autonomous run, the queue holds three items: a choice between two database index designs, a question whether to announce a deprecation to users (outward), and a request to approve a paid evaluation batch with its estimated cost. The person answers all three in ten minutes. The agents pick up the rulings the next morning and continue without asking anything else.

## After the ruling

The agent copies the ruling into the decision record in the person's words, marks the item resolved, and acts on it. The person's wording is kept because a paraphrase can drift ("use option A" becomes "use A for now"), and later agents read the record, not the chat.

### Example: a ruling that refines an option

The person writes: "A, but keep the old endpoint working until the end of the month." The agent records the ruling verbatim, adds the end-of-month removal as a new task with a date, and implements option A with the old endpoint intact. Had it paraphrased the ruling as "A", the condition would have been lost.

## Examples from my own harness

- Every question that survives the authority test is written as a held decision, formatted with context, options and a recommendation, and followed by a plain explanation of the trade-off.
- Each session keeps one aggregate review document listing everything I must review for its current goal, compiled from the held items and checked before it is shown to me.
- A run that cannot finish ends with one exit report naming every held item by path, instead of questions spread across the chat.

To start: one file, `decisions.md`, in the repository root, with the template above, and an instruction in your context file to write every question for you there.

## Pitfalls

- A queue nobody reads is a slower way to block. Tell the person when it has items, once, at the end of a run.
- A held decision without a recommendation pushes the analysis back to the person.
- Questions that the authority test would have resolved fill the queue and hide the real ones. Run the test before writing the item.

## Sources

- Held decisions and the review queue: from the author's own harness, described conceptually.

{% include page_nav.html %}
