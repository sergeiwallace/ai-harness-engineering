---
title: "Alignment with the original request"
parent: "Human gates, decisions and alignment"
nav_order: 2
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Alignment with the original request

Part of [Human gates, decisions and alignment]({{ '/sections/14-human-gates/' | relative_url }}).

| Term | Meaning |
|---|---|
| Alignment | the work an agent delivers matches what the person asked for, all of it, in the sense they meant |
| Drift | the loss of that match as work passes between hands |

Drift happens at each hand-off, because each one is a summary that can drop something:

1. person to plan;
2. plan to brief;
3. brief to sub-agent;
4. sub-agent back up as a report.

## Contents

- [How drift happens](#how-drift-happens)
- [Keep the original words](#keep-the-original-words)
  - [When the person adds or changes something](#when-the-person-adds-or-changes-something)
- [Carry the request into every brief](#carry-the-request-into-every-brief)
- [The drift check](#the-drift-check)
  - [Who runs it](#who-runs-it)
  - [As a gate in the system](#as-a-gate-in-the-system)
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

### When the person adds or changes something

Requests change during a task. Keep the original and append each change as its own dated entry, so the history stays readable and each entry can be checked:

```text
request.txt          the original, never edited
amendments/
  2026-11-09T14:10Z.txt   "Also add a CSV option to the export."
  2026-11-09T16:42Z.txt   "Drop the dark mode toggle for now."
request.sha256       hashes of the original and every amendment
```

| Case | What the drift check uses |
|---|---|
| The person adds a requirement | the original plus the amendment |
| The person removes one | the amendment, which records that the person, not an agent, dropped it |
| An agent proposes a change and the person agrees | the ruling in the held decision, copied verbatim as an amendment |
| An agent narrows something on its own | nothing: the check fails |

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

The drift check compares the result with the original request, and any amendments, before the work is called complete. It has two parts:

1. Extract the requirements the person named, one per line, from the verbatim request.
2. For each, find the evidence in the result that it was delivered, or an explicit escalation where the person agreed to change it.

A requirement with neither fails the check. So does any agent decision that drops, narrows or contradicts a named requirement, even if the decision is recorded and reasoned: those decisions belong to the person ([Decision authority and escalation]({{ '/sections/14-human-gates/decision-authority.html' | relative_url }}), question 3).

```text
Request requirement           Evidence                           Status
dark mode toggle              settings page, toggle test passes  delivered
export button                 none                               FAIL
                              agent decision "defer export"      not the agent's call
```

### Who runs it

| Runner | Strength | Weakness |
|---|---|---|
| The agent that did the work | cheap, knows the evidence | reads its own result generously |
| A second agent, given only the verbatim request and the result | independent reading | costs a second run |
| A gate script over a requirements file | runs every time, cannot be skipped | needs the requirements extracted first |

### As a gate in the system

An agent, or the person, extracts the requirements into a file once. A completion gate then refuses to mark the task done while any requirement lacks evidence:

```yaml
# requirements.yaml, extracted from request.txt and its amendments
- id: toggle
  text: "a dark mode toggle"
  evidence: "tests/test_settings.py::test_dark_mode_toggle"
- id: export
  text: "an export button"
  evidence: null          # nothing yet: the gate fails
```

```python
import sys
import yaml

missing = [r["id"] for r in yaml.safe_load(open("requirements.yaml")) if not r.get("evidence")]
if missing:
    sys.exit(f"drift check failed: no evidence for {', '.join(missing)}. "
             "Deliver them, or open a held decision for the person.")
print("drift check passed")
```

The refusal names the two allowed routes: deliver the requirement, or ask the person.

![The original request, stored with a hash, is carried into each brief. The result is compared against the original by a drift check, and a dropped requirement is escalated.]({{ '/assets/images/14-human-gates/alignment-chain.svg' | relative_url }}){: style="width:100%"}

*The request travels unchanged with the work, and the result is compared with it before the task is called done.*

## Worked examples

### Example 1: the deferred button

1. The settings page ships with the toggle but not the export button; the agent judged the endpoint "a follow-up".
2. The drift check fails the export row.
3. The agent writes a held decision:
   - build the endpoint now, about half a day;
   - ship the page without the button and file the button as a dated task.
4. The person picks the first.

### Example 2: a narrowing caught in a brief

1. A coordinator writes a brief: "export for admin users".
2. The sub-agent compares it with the verbatim block, which names no audience.
3. It asks the coordinator before building.
4. The coordinator had narrowed it to avoid a permissions question. It restores "all users" and adds the permissions question to the review queue.

### Example 3: a substitution that was better, and still escalated

| | |
|---|---|
| Request | a CSV export |
| Agent's view | an Excel file would serve the users better |
| What it builds | CSV, as asked |
| What it adds | a one-line item in the review queue proposing Excel |
| Who decides the switch | the person |

## Examples from my own harness

- The founding request of every task is stored verbatim with a hash and a fidelity tier, and the tier decides how strictly the result is checked against it.
- Every brief carries the verbatim request in a fixed block, followed by framing explicitly marked non-authoritative, and an alignment obligation.
- A gate fails any agent decision that drops, narrows or contradicts an explicit requirement. The only ways through are to keep the requirement or to escalate it.

To start: store the original request in a file with its hash, have the brief template include it verbatim, and add a completion check that lists each requirement with its evidence and refuses to mark the task done while one is missing. A context-file line asking the agent to preserve the request is the second layer.

## Pitfalls

- Storing the request but carrying only a summary into briefs defeats the purpose. The verbatim text must travel.
- A drift check written by the same agent from the same summary will agree with itself. Extract the requirements from the verbatim text.
- Not every word is a requirement. "Maybe also" and "if easy" are suggestions; the check marks them as such rather than failing on them.
- Editing the original request in place to match what was built hides the drift. Keep the original and record each change as an amendment.
- A requirements file extracted once can itself drop something. Have a second agent extract it from the verbatim text and compare the two lists.

## Sources

- The verbatim request, the brief format and the drift gate: from the author's own harness, described conceptually.

{% include page_nav.html %}
