---
title: "Decision authority and escalation"
parent: "Human gates, decisions and alignment"
nav_order: 1
has_children: true
has_toc: false
redirect_from:
  - /sections/14-human-gates/detail.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Decision authority and escalation

Part of [Human gates, decisions and alignment]({{ '/sections/14-human-gates/' | relative_url }}). Sub-page: [Held decisions and the review queue]({{ '/sections/14-human-gates/held-decisions.html' | relative_url }}).

Decision authority is the rule set that says which decisions an agent makes alone, which it makes and records, and which go to a person. Escalation is the act of sending a decision up, to a stronger model or to the person. A harness with clear authority lets agents run for hours without stalling, because they do not stop on calls they own, and without overreaching, because they stop on calls they do not.

## Contents

- [The authority test](#the-authority-test)
- [Investigate before you escalate](#investigate-before-you-escalate)
- [Who an agent escalates to](#who-an-agent-escalates-to)
  - [What an escalation message contains](#what-an-escalation-message-contains)
- [Recording agent decisions](#recording-agent-decisions)
- [Worked examples](#worked-examples)
- [Examples from my own harness](#examples-from-my-own-harness)
- [Pitfalls](#pitfalls)
- [Sources](#sources)

## The authority test

The authority test is a short set of questions an agent answers about a decision before it acts. It replaces a long list of rules with a few properties that every rule on the list was standing in for.

| # | Question | If yes | If no |
|---|---|---|---|
| 1 | Did the person claim this decision? | the person decides | next question |
| 2 | Is it outward, destructive, spending or architectural? | the person decides, unless a program has proved it reversible | next question |
| 3 | Would it drop, narrow or change something the person asked for? | the person decides | next question |
| 4 | Is it a design call beyond this model's tier? | a stronger model decides | the agent decides and records why |

![A vertical flow chart of the authority test. A decision to make, before the agent acts, meets four questions in order. Question 1: did the person claim it? Question 2: is it outward, destructive, spending or architectural? Question 3: does it drop or narrow what the person asked for? A yes to any of these sends it to the person, who decides; for question 2, unless a program has proved the action reversible. Question 4: is it a design call beyond this model's tier? A yes sends options and context to a stronger model, which decides. If all four answers are no, the agent decides and records why.]({{ '/assets/images/diagrams/14-human-gates/authority-test-flow.svg' | relative_url }})
*Figure: the authority test. A yes to any of the first three questions sends the decision to the person; the fourth sends a design call up a model tier instead. Made with archify.*

*The authority test on one design question. A yes to any of the first three questions sends the decision to the person; the fourth sends a design call up a model tier instead.*

![Decision path for any single action: outward, destructive, spending or architectural actions go to the person through a held decision; otherwise the agent investigates, decides and records why, unless a gate still applies.]({{ '/assets/images/14-human-gates/gate-flow.svg' | relative_url }}){: style="width:100%"}

*The same test as a decision path.*

### Question 1: did the person claim it?

Anything the person marked as theirs is theirs, however small.

- Signs of a claim: "I want to choose", "check with me before", an item marked pending.
- Example: a person who wrote "I want to choose the colour scheme myself" is asked about the colour scheme, though it is cosmetic.

### Question 2: is it outward, destructive, spending or architectural?

These four kinds are hard or impossible to undo, or other people see them:

| Kind | Example | Why it cannot be undone |
|---|---|---|
| Outward | a sent message, a public comment, a published page | others have seen it |
| Destructive | deleting data, a table, a cloud resource | the data may be gone |
| Spending | a paid batch, rented compute | a revert does not refund it |
| Architectural | a new dependency, a public interface, a schema | every later change builds on it |

Reversibility is the underlying test. Deleting a branch that is fully merged and still on the remote is a deletion, but nothing is lost. A harness can exempt such proved-safe cleanup from the gate:

1. A program checks the proof, for example that every commit on the branch is on the remote main.
2. Only when the check passes does the deletion run (`check && delete`).
3. The agent's own statement that the branch is merged is never the proof.

### Question 3: would it change what the person asked for?

An agent may decide how to deliver a requirement. Delivering less is the person's call:

- Dropping a feature ("defer the export button to a follow-up").
- Narrowing its scope ("only for admins").
- Changing its meaning ("CSV" becomes "Excel" because the agent judged it better).

This is the question most often missed, because from inside the task the decision looks technical. [Alignment with the original request]({{ '/sections/14-human-gates/alignment-and-drift.html' | relative_url }}) covers the drift check that enforces it.

### Question 4: is this the right model to decide?

Some decisions are within the agent's remit but beyond a small model's judgement:

- a design trade-off between two schemas or two module boundaries;
- a security question, such as whether a token may be logged;
- an ambiguous requirement with two reasonable readings.

A harness can set a floor, such as "only the strongest models may resolve a design decision", and enforce it in the launcher: a smaller model's design question is routed to a stronger session, which decides and records it. The person is not involved.

## Investigate before you escalate

A blocked agent investigates first and asks only if a gate still applies. Investigation means:

1. Read the error in full and find the actual cause.
2. Check the docs, the code and the harness for an existing route (an automated login, a script, a documented fix).
3. Try the reversible fix.
4. Re-run the authority test with what you now know.

Most blockers dissolve at step 2 or 3. When one does not, the investigation becomes the context section of the question, so the person does not have to redo it.

### Example: a permission error that was not one

1. An agent calling a cloud API gets "access denied".
2. It drafts a question asking the person to grant a permission.
3. Investigating first, it prints which identity the call used and finds the machine's default credentials, not the named profile.
4. It re-runs with `--profile` set; the call succeeds.
5. No question is needed. An error that reads like a missing grant is often a wrong identity, so the harness's wrapper now always passes the profile.

## Who an agent escalates to

Escalation has more than one destination:

| Situation | Escalate to |
|---|---|
| A design decision, and the agent runs on a small or fast model | A stronger model, for example the session that launched the agent |
| A blocker the agent cannot investigate further, inside its remit | Its launcher, the coordinator that gave it the task |
| A decision the authority test reserves for the person | The person, through a held decision |
| A question about another agent's work | That agent's owner, by message |

Sending everything to the person is the common failure. A sub-agent that hits a design question asks its coordinator first; the coordinator either decides or escalates once, with the context already gathered.

### What an escalation message contains

```text
To: coordinator
Task: export endpoint for large files
Decision needed: stream the file, or split it into 500 MB parts?
Investigated: the current endpoint loads the whole file; 2 GB files fail with out-of-memory.
Options:
  A. Stream with a chunked response. One function changes; clients need no change.
  B. Split into parts. Clients must reassemble; simpler server code.
Recommendation: A, because no client change is needed.
Blocking: yes, the endpoint work waits on this.
```

- The receiver can decide from the message alone, without opening the agent's transcript.
- "Blocking" tells the receiver whether the sender is idle while it waits.
- If the receiver escalates further, it forwards the message unchanged and adds its own view below it.

## Recording agent decisions

When the agent decides, it writes down the decision and the reason. A record turns "the agent did something I did not expect" into "the agent chose B over A for this reason", which the person can overrule in one line.

| Where | Good for |
|---|---|
| A section of the pull request | decisions tied to one change; the reviewer sees them with the diff |
| The plan or design document | decisions that shape later work |
| A decision log file | decisions across many changes, searchable later |

Record decisions a reviewer could reasonably question: a format, a dependency avoided, a trade-off. Skip the trivial ones, such as a variable name.

```markdown
### Decision: store exports as CSV, not Parquet (agent, 2026-11-09)

Context: the export feature needs a file format. The request did not name one.
Options: CSV (opens in any spreadsheet) or Parquet (smaller, typed).
Chosen: CSV. The stated users are analysts who open exports in a spreadsheet.
Reversible: yes, the writer is one function.
```

## Worked examples

### Example 1: a new dependency

| Step | What happens |
|---|---|
| Situation | an agent adding date parsing wants a third-party library |
| Test | question 2: a new dependency is architectural |
| Outcome | it writes a held decision |
| Options | the library (maintained, adds a package); the standard library (more code, no new package) |
| Recommendation | the standard library, because only two date formats appear in the data |

### Example 2: a flaky test blocks a merge

| Step | What happens |
|---|---|
| Situation | a flaky test blocks a merge |
| Test | skipping changes nothing the person asked for, but the harness rule is "fix every test failure before shipping" |
| Outcome | the agent investigates, finds a race in the test's setup, fixes it and records the cause |
| Question to the person | none |

### Example 3: a cheaper model with a design question

| Step | What happens |
|---|---|
| Situation | a small model writing a migration finds two ways to normalise the schema |
| Test | question 4: a design call |
| Outcome | it sends both options to its coordinator, on a stronger model |
| Result | the coordinator decides and records it; the person never sees it |

### Example 4: a request that cannot be met as written

| Step | What happens |
|---|---|
| Situation | the person asked for a report "updated every minute"; the source rate-limits to one call every five minutes |
| Test | question 3: a five-minute refresh would narrow the request |
| Outcome | a held decision with the measured limit |
| Options | a five-minute refresh; or a page that reloads every minute from a cache, showing the data's age |

## Examples from my own harness

- A decision procedure every decision and blocker passes before it may reach me. It runs the investigation and the authority test, decides when the agent has authority, and re-runs after unblocking. A decision it resolves is never put to me for review.
- A model-tier floor: only the strongest models may resolve a design decision; smaller models ask the strongest session on the same machine.
- Every escalated decision becomes a held decision in a review queue ([Held decisions and the review queue]({{ '/sections/14-human-gates/held-decisions.html' | relative_url }})).

To start: build the four questions into the workflow the agent runs before it asks you anything, for example a skill or a pull request template with a section per question, and have a check refuse a pull request whose decision log is empty. A context-file line pointing at the questions is the second layer.

## Pitfalls

- An authority test without investigation produces well-formatted questions the agent could have answered itself.
- "Reversible" needs proof. An agent's claim that a branch is merged is not proof; a program checking the remote is.
- A decision record written after the fact tends to justify the choice. Ask for the options to be written before the choice.
- A question asked in the middle of a chat is lost when the session ends. Route every question for the person through the review queue.
- An agent that asks again after the person answered has not recorded the ruling. Read the decision file before asking.
- A coordinator that forwards every sub-agent question to the person adds a hop and no judgement. It decides what it has the authority to decide.

## Sources

- The authority test, the tier floor and the decision procedure: from the author's own harness, described conceptually.

{% include page_nav.html %}
