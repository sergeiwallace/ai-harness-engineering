---
title: "Alignment with the original request"
parent: "Human gates, decisions and alignment"
nav_order: 2
---

{% include page_nav.html %}

# Alignment with the original request

As of 2026-11-09.

Part of [Human gates, decisions and alignment]({{ '/sections/14-human-gates/' | relative_url }}).

Alignment here means something narrow and practical: the work an agent delivers matches what the person asked for, all of it, in the sense they meant. Drift is the loss of that match as work passes from the person to a plan, from a plan to a brief, from a brief to a sub-agent, and from a sub-agent back up as a report. Each hand-off is a summary, and each summary can drop something.

## Contents

- [How drift happens](#how-drift-happens)
- [Keep the original words](#keep-the-original-words)
- [Carry the request into every brief](#carry-the-request-into-every-brief)
- [The drift check](#the-drift-check)
- [Worked examples](#worked-examples)
- [Examples from my own harness](#examples-from-my-own-harness)
- [Pitfalls](#pitfalls)

## How drift happens

![Sequence diagram. The person asks the lead agent for A, B and C. The lead briefs a sub-agent with A and B plus the verbatim request; the sub-agent reports A and B done. The lead runs the drift check against the request, which fails because C is missing. The lead writes a held decision on C; the person says build C now. In a second pass the lead briefs C, the sub-agent reports it done, the drift check passes with all requirements kept, and the lead reports done with evidence.]({{ '/assets/images/14-human-gates/drift-sequence.svg' | relative_url }}){: style="width:100%"}

*A dropped requirement, caught by the drift check and restored in a second pass.*

Drift has a few common shapes:

| Shape | What it looks like |
|---|---|
| Omission | A requirement is left out of a brief because the writer thought it was covered elsewhere |
| Narrowing | "Export for all users" becomes "export for admins" to make the first version easier |
| Substitution | The agent builds something close to the request that it judged better |
| Deferral | A requirement is moved to "a follow-up" that nobody files |
| Paraphrase loss | A condition ("until the end of the month") disappears when the request is restated |

Each shape is a reasonable local decision by the agent that made it. Tests and reviews miss them because the tests check what was built and the reviewer compares the diff with the plan; the dropped requirement appears only in the person's words.

## Keep the original words

The first defence is to store the person's request verbatim and treat it as the reference for the whole task. Paraphrases are useful for planning, but the verbatim text is what the result is checked against.

A content hash makes the stored text tamper-evident: any agent can recompute the hash and confirm it is reading the original text.

```bash
# store the request once, at the start of the task
printf '%s' "$REQUEST" > request.txt
sha256sum request.txt > request.sha256

# any later agent can confirm it holds the original
sha256sum --check request.sha256
```

## Carry the request into every brief

Every brief to a sub-agent carries the verbatim request in a fixed block, separate from the task-specific framing. The framing tells the agent its part; the verbatim block tells it the whole, so it can notice when its part conflicts with the whole.

```markdown
## Original request (verbatim, sha256 3f9a...e21c)

    Add a settings page with a dark mode toggle and an export button.

## Your part (framing, not authoritative)

Build the export button and its endpoint. Another agent builds the toggle.

## Alignment obligation

Preserve the original request. If your part cannot be done as asked,
say so and stop; do not deliver less without escalating.
```

The label "Framing, not authoritative" tells the agent that if the framing and the original conflict, the original wins and the conflict is reported.

## The drift check

The drift check compares the result with the original request before the work is called complete. It has two parts.

1. Extract the requirements the person named, one per line, from the verbatim request.
2. For each, find the evidence in the result that it was delivered, or an explicit escalation where the person agreed to change it.

A requirement with neither fails the check. So does any agent decision that drops, narrows or contradicts a named requirement, even if the decision is recorded and reasoned: those decisions belong to the person ([Decision authority and escalation]({{ '/sections/14-human-gates/decision-authority.html' | relative_url }}), question 3).

```text
Request requirement           Evidence                           Status
dark mode toggle              settings page, toggle test passes  delivered
export button                 none                               FAIL
                              agent decision "defer export"      not the agent's call
```

The check can be run by an agent other than the one that did the work, which helps, because the author tends to read its own result generously. It can also be a gate in the harness that refuses to mark the task complete while any row fails.

![The original request, stored with a hash, is carried into each brief. The result is compared against the original by a drift check, and a dropped requirement is escalated.]({{ '/assets/images/14-human-gates/alignment-chain.svg' | relative_url }})

## Worked examples

### Example 1: the deferred button

The settings page ships with the toggle but not the export button, because the agent judged the endpoint "a follow-up". The drift check fails the export row. The agent writes a held decision: build the endpoint now (half a day), or ship the page without the button and file the button as a dated task. The person picks the first.

### Example 2: a narrowing caught in a brief

A coordinator writes a brief: "export for admin users". The sub-agent compares it with the verbatim block ("an export button", no audience named), notices the narrowing, and asks the coordinator. The coordinator had narrowed it to avoid a permissions question; it restores "all users" and adds the permissions question to the queue.

### Example 3: a substitution that was better, and still escalated

The person asked for a CSV export. The agent believes an Excel file would serve the users better. It still builds CSV, and adds a one-line note to the queue proposing Excel as an option. The person may agree, but the switch is theirs to make.

## Examples from my own harness

- The founding request of every task is stored verbatim with a hash and a fidelity tier, and the tier decides how strictly the result is checked against it.
- Every brief carries the verbatim request in a fixed block, followed by framing explicitly marked non-authoritative, and an alignment obligation.
- A gate fails any agent decision that drops, narrows or contradicts an explicit requirement. The only ways through are to keep the requirement or to escalate it.

To start: paste the original request at the top of your plan document, and before you call a task done, ask the agent to list each requirement from that text with the evidence it was delivered.

## Pitfalls

- Storing the request but carrying only a summary into briefs defeats the purpose. The verbatim text must travel.
- A drift check written by the same agent from the same summary will agree with itself. Extract the requirements from the verbatim text.
- Not every word is a requirement. "Maybe also" and "if easy" are suggestions; the check should mark them as such rather than fail on them.

## Sources

- The verbatim request, the brief format and the drift gate: from the author's own harness, described conceptually.

{% include page_nav.html %}
