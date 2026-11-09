---
title: "The original-request block"
parent: "Sub-agent briefs"
grand_parent: "Templates, prompt stubs and injected prompts"
nav_order: 1
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# The original-request block

The original-request block is a fixed block at the top of every brief that holds the person's words exactly as they were written. It exists because requests drift: each agent in a chain restates the one before, and a requirement can disappear while every check stays green. This page covers the drift, the incident that showed it, how the block is stored and checked, and how an agent uses it.

![A sequence across five participants. The person asks for A, B and C, with C using injection. The lead stores the words and hash in the request record. The lead sends a brief with a tidied summary to the launch guard, which compares the hash and refuses it. The lead copies the block unchanged and the guard launches the worker. The worker checks whether its plan keeps C, finds it would drop a named requirement, and tells the lead, which asks the person whether to keep C with injection or change the ask.]({{ '/assets/images/07-templates-and-stubs/request-guard.png' | relative_url }}){: style="width:100%"}

*The original-request block stops a dropped requirement. Built with archify.*

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Drift

### How it happens

<!-- IMAGE NEEDED: the telephone game (Chinese whispers), a message changing as it passes from person to person; search: telephone game illustration, chinese whispers drawing, message distortion chain -->

A request passes through several hands before anyone builds it:

1. The person asks a lead agent.
2. The lead writes a plan in its own words.
   - It shortens the request and moves part of it to a later phase.
3. A planner splits the plan into tasks.
   - It works from the lead's plan, so the later phase can drop out.
4. A worker builds one task from its brief.
   - The brief is the third restatement of the request.

Each restatement looks reasonable on its own. The table shows how they add up.

| Hand | What it wrote | What changed |
|---|---|---|
| Person | "Add A, B and C. C should use injection." | |
| Lead agent | "Build A and B; C uses injection, later phase." | C moved to "later" |
| Planner | "Phase 1: A and B." | C left the plan |
| Worker | Builds A and B, all tests pass | C never existed for it |

### Why checks miss it

Every gate in the chain checks the work against the brief it received:

| Gate | Checks against | Has the person's words? |
|---|---|---|
| Worker's tests | The worker's brief | No |
| Planner's plan review | The lead's summary | No |
| Lead's acceptance | Its own plan | No |

## The incident

A person asked for a feature and named prompt injection as one of its mechanisms. Two hands later, an agent's decision record said "injection is not used", and the work went ahead without it. Every gate passed. The gap surfaced only when the person read the result and asked where the injection was.

Three things were missing, and the fix supplies each:

1. The person's words were not in the later briefs, so no agent could compare against them.
2. Nothing stopped an agent from deciding to drop a named requirement.
3. Nothing at launch noticed that a brief carried a paraphrase instead of the words.

## The block

### What it contains

```markdown
## Original request (verbatim, do not edit)
> add a table of contents on the landing page and the root README.md
record: asks/site-restructure.md  sha256: 9f2c...e41a
```

The quoted text is the person's message, byte for byte, typos included. The record line names where the words are stored and the hash of the stored copy.

### Where it sits

- At the very top of the brief, above the objective.
  - It is the first thing a sub-agent reads.
  - A reviewer cannot miss it.
- In every brief the lead writes.
- In every brief a sub-agent writes for its own helpers, copied again.
  - It never shortens on the way down the tree.

## Storage and the launch guard

The figure at the top of this page shows the whole path. In steps:

1. The person's words are stored once, with a hash.
2. The block is copied, unchanged, into every brief.
3. At launch, the guard compares the block with the record. A mismatch refuses the launch, with the reason.
4. The agent starts, and checks its result against the words before it reports.

### Storing the words

1. The first agent that receives the request writes it to a record file.
   - Byte for byte, typos included.
2. It computes the hash of the record.
3. From then on the record is the source; briefs carry copies.

```text
asks/dates-tz.md
sha256 41ab...07c2
> make every date in the API responses carry an explicit timezone
```

### The guard

Before a sub-agent starts:

1. The launch guard reads the brief and finds the original-request block.
2. It hashes the quoted text and compares it with the record's hash.
3. It refuses the launch, with the reason, when:
   - The brief has no block.
   - The block's text differs from the record.
   - The record named in the block does not exist.
4. Otherwise the launch goes ahead.

This is the enforcing layer: the agent does not have to remember to copy the block correctly, because a wrong copy never launches.

#### Worked example: a refused launch

```text
launch refused: original-request block does not match its record
  record:  asks/dates-tz.md  sha256 41ab...07c2
  brief:   sha256 9d10...aa3f  (text differs at line 1)
  fix:     copy the block from the record unchanged
```

<details markdown="1">
<summary>Why a hash and not a text comparison by eye</summary>

An agent restating the request does not think of it as editing. It tidies a typo, drops a clause that "is covered elsewhere", or merges two sentences. Each looks harmless to a reader. A hash changes when any character of the text changes, so the guard catches each of these edits.

</details>

## How an agent uses the block

### During the work

The agent uses the block as a reference to check against.

- At each decision point, it asks whether the choice keeps everything the person named.
- When a plan step names a requirement, it compares the wording with the block.
- When it starts helpers, it copies the block into their briefs unchanged.

### At a decision that would drop something

An agent may decide many things on its own. It may not decide to drop, narrow or contradict a requirement the person stated. When a choice would do that, the agent keeps the requirement or takes the question back to the person, with the conflict stated.

| Decision | The agent may | The agent may not |
|---|---|---|
| Which library to use for C | Decide | |
| Whether to build C in this phase or a later one | Decide, if the later phase is scheduled and recorded | Drop C without saying so |
| Whether C uses injection | | Decide "not used" when the person named it |

### At the end

The final report compares the result with the block, one named requirement at a time.

| Requirement in the block | Status | Evidence |
|---|---|---|
| Table of contents on the landing page | Met | Page URL and heading |
| Table of contents in the root README.md | Met | Commit and line |

## Edge cases

| Case | How the block handles it |
|---|---|
| The person adds a correction later in the session | The correction is appended to the record as a new, dated entry with its own hash; earlier words stay as they were |
| The request spans several messages | Each message goes into the record in order, each quoted exactly |
| The request is long | The whole text still goes in; a summary may sit next to it, labelled as a summary and never in place of the words |
| The launcher adds its own framing (scope, paths, deadline) | The framing goes in a separate section, labelled as not authoritative, below the block; where the two conflict, the words win and the conflict is reported |
| The person changes their mind | Only the person changes the record; an agent that thinks the ask should change says so and waits |
| A sub-agent needs only part of the request | It still gets the whole block, so it can see what its part is for |

### Worked example: words plus framing

```markdown
## Original request (verbatim, do not edit)
> add a table of contents on the landing page and the root README.md
record: asks/site-restructure.md  sha256: 9f2c...e41a

## Framing from the launcher (not authoritative)
You own the README part only. Another agent owns the landing page.
Deadline 18:00. If this framing conflicts with the request above, follow the
request and report the conflict.
```

## Pitfalls

| Pitfall | Effect | Fix in the system |
|---|---|---|
| The block is "tidied" on the way down | A requirement changes with every check green | The launch guard compares the hash |
| Framing is mixed into the quoted words | A launcher's scope reads as the person's ask | A separate, labelled framing section |
| The record is overwritten by a correction | The original ask is lost | Append-only record, one hash per entry |
| An agent drops a requirement it judged unnecessary | The person finds out at the end | A rule that drop decisions go back to the person, checked in the final report |

## Related pages

- [Sub-agent briefs]({{ '/sections/07-templates-and-stubs/briefs.html' | relative_url }}), the template the block sits at the top of.
- [Human gates]({{ '/sections/14-human-gates/' | relative_url }}), where a decision goes when the agent may not make it.

## Sources

- The original-request block, its record, hash and launch guard are from the author's own harness, described conceptually. The incident is described at the conceptual level. The examples are illustrations written for this page.

{% include page_nav.html %}
