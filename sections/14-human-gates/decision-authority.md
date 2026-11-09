---
title: "Decision authority and escalation"
parent: "Human gates, decisions and alignment"
nav_order: 1
has_children: true
has_toc: false
redirect_from:
  - /sections/14-human-gates/detail.html
---

{% include page_nav.html %}

# Decision authority and escalation

As of 2026-11-09.

Part of [Human gates, decisions and alignment]({{ '/sections/14-human-gates/' | relative_url }}). Sub-page: [Held decisions and the review queue]({{ '/sections/14-human-gates/held-decisions.html' | relative_url }}).

Decision authority is the rule set that says which decisions an agent makes alone, which it makes and records, and which go to a person. Escalation is the act of sending a decision up, to a stronger model or to the person. A harness with clear authority lets agents run for hours without stalling, because they do not stop on calls they own, and without overreaching, because they stop on calls they do not.

## Contents

- [The authority test](#the-authority-test)
- [Investigate before you escalate](#investigate-before-you-escalate)
- [Who an agent escalates to](#who-an-agent-escalates-to)
- [Recording agent decisions](#recording-agent-decisions)
- [Worked examples](#worked-examples)
- [Examples from my own harness](#examples-from-my-own-harness)
- [Pitfalls](#pitfalls)
- [Sources](#sources)

## The authority test

The authority test is a short set of questions an agent answers about a decision before it acts. It replaces a long list of rules with a few properties that every rule on the list was standing in for.

![Sequence diagram. A small-model agent asks the authority test three questions in turn: claimed by the person, outward or architectural, drops a requirement. Each answer is no, but the last notes it is a design call. The agent sends two options and context to the coordinator on the strongest model, which returns a decision and reason. The agent writes the decision log into the pull request for the person, who may overrule later.]({{ '/assets/images/14-human-gates/authority-test-sequence.svg' | relative_url }}){: style="width:100%"}

*The authority test on one design question. A yes to any of the first three questions sends the decision to the person; the fourth sends a design call up a model tier instead.*

![Decision path for any single action: outward, destructive, spending or architectural actions go to the person through a held decision; otherwise the agent investigates, decides and records why, unless a gate still applies.]({{ '/assets/images/14-human-gates/gate-flow.svg' | relative_url }}){: style="width:100%"}

*The same test as a decision path.*

### Question 1: did the person claim it?

Anything the person marked as pending their decision is theirs, however small. A person who wrote "I want to choose the colour scheme myself" gets asked about the colour scheme even though it is cosmetic.

### Question 2: is it outward, destructive, spending or architectural?

These four kinds share one property: they are hard or impossible to undo, or other people see them. An outward action (a sent message, a public comment) cannot be unsent. A destructive one may lose data. Spending cannot be refunded by a revert. An architectural choice constrains every later change, so undoing it later costs far more than asking now.

Reversibility is the underlying test. Deleting a branch that is fully merged and still on the remote is technically a deletion, but nothing is lost. A harness can exempt such proved-safe cleanup from the gate, with the proof checked by a program rather than asserted by the agent.

### Question 3: would it change what the person asked for?

An agent may decide how to deliver a requirement. It may not decide to deliver less than was asked. Dropping a feature, narrowing its scope ("only for admins"), or changing its meaning without saying so are the person's calls. This is the question most often missed, because the decision usually looks technical from inside the task. [Alignment with the original request]({{ '/sections/14-human-gates/alignment-and-drift.html' | relative_url }}) covers the check that enforces it.

### Question 4: is this the right model to decide?

Some decisions are within the agent's remit but beyond a small model's judgement: a design trade-off, a security question, an ambiguous requirement. A harness can set a floor, such as "only the strongest models may resolve a design decision", and route a weaker model's design question to a stronger one rather than to the person.

## Investigate before you escalate

A blocked agent investigates first and asks only if a gate still applies. Investigation means:

1. Read the error in full and find the actual cause.
2. Check the docs, the code and the harness for an existing route (an automated login, a script, a documented fix).
3. Try the reversible fix.
4. Re-run the authority test with what you now know.

Most blockers dissolve at step 2 or 3. When one does not, the investigation becomes the context section of the question, so the person does not have to redo it.

### Example: a permission error that was not one

An agent calling a cloud API gets "access denied" and drafts a question asking the person to grant a permission. Investigating first, it notices the call used the machine's default credentials instead of the named profile. It re-runs with the profile, the call succeeds, and no question is needed. An error message that reads like a missing grant is often a wrong identity.

## Who an agent escalates to

Escalation has more than one destination:

| Situation | Escalate to |
|---|---|
| A design decision, and the agent runs on a small or fast model | A stronger model, for example the session that launched the agent |
| A blocker the agent cannot investigate further, inside its remit | Its launcher, the coordinator that gave it the task |
| A decision the authority test reserves for the person | The person, through a held decision |
| A question about another agent's work | That agent's owner, by message |

Sending everything to the person is the common failure. A sub-agent that hits a design question should ask its coordinator first; the coordinator either decides or escalates once, with the context already gathered.

## Recording agent decisions

When the agent decides, it writes down the decision and the reason. The record can be a line in the plan document, a section in the pull request, or an entry in a decision log. A record turns "the agent did something I did not expect" into "the agent chose B over A for this reason", which the person can overrule in one line.

```markdown
### Decision: store exports as CSV, not Parquet (agent, 2026-11-09)

Context: the export feature needs a file format. The request did not name one.
Options: CSV (opens in any spreadsheet) or Parquet (smaller, typed).
Chosen: CSV. The stated users are analysts who open exports in a spreadsheet.
Reversible: yes, the writer is one function.
```

## Worked examples

> Each example below names the question of the authority test that decided it.

### Example 1: a new dependency

#### The situation

An agent adding date parsing wants a third-party library.

#### The test

Question 2 says a new dependency is architectural, so the agent asks.

#### The question it wrote

Options: the library (well maintained, adds a package) or the standard library (more code, no new package). It recommends the standard library, because only two formats appear in the data.

### Example 2: a test that should be skipped

A flaky test blocks a merge. Skipping it changes nothing the person asked for, but it lowers the safety net, and the harness rule is "fix every test failure before shipping". The agent investigates, finds a race in the test's setup, fixes it and records the cause. No question is needed.

### Example 3: a cheaper model with a design question

A small model writing a migration finds that the schema could be normalised two ways. Question 4 says this is a design call, so the small model sends the options to its coordinator, which runs on a stronger model. The coordinator decides and records the decision; the person never sees it.

### Example 4: a request that cannot be met as written

The person asked for a report "updated every minute", but the source system rate-limits to one call every five minutes. Delivering every five minutes would narrow the request, so question 3 sends it to the person, with the measured limit and two options (five-minute refresh, or a cached feed with a staleness note).

## Examples from my own harness

- A decision procedure every decision and blocker passes before it may reach me. It runs the investigation and the authority test, decides when the agent has authority, and re-runs after unblocking. A decision it resolves is never put to me for review.
- A model-tier floor: only the strongest models may resolve a design decision; smaller models ask the strongest session on the same machine.
- Every escalated decision becomes a held decision in a review queue ([Held decisions and the review queue]({{ '/sections/14-human-gates/held-decisions.html' | relative_url }})).

To start: add the four questions above to your context file, and ask the agent to record every decision it makes alone in the pull request description.

## Pitfalls

- An authority test without investigation produces well-formatted questions the agent could have answered itself.
- "Reversible" needs proof. An agent's claim that a branch is merged is not proof; a program checking the remote is.
- A decision record written after the fact tends to justify the choice. Ask for the options to be written before the choice.

## Sources

- The authority test, the tier floor and the decision procedure: from the author's own harness, described conceptually.

{% include page_nav.html %}
