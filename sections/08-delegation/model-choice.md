---
title: "Model choice"
parent: "Agent profiles"
grand_parent: "Agent profiles, sub-agents and delegation"
nav_order: 1
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Model choice

Delegation lets each part of a task run on a model suited to it. A common split, the one figure 1 shows:

1. The lead keeps the design decisions on the strongest model.
2. Volume work, such as searching, goes to a cheaper model whose output the lead can check.
3. Implementation of a decided brief goes to a mid-tier model.
4. Review goes to a model from a different family. This page covers the tiers, how to set the model and effort, a worked task across four tiers, and when to break the default.

![A sequence across four participants. The lead, on the strongest tier, asks a searcher on the cheapest tier to list callers of parse_date and gets a nine-row table. The lead decides the design and briefs an engineer on the mid tier to fix the three callers without a timezone; the engineer returns a commit and a test line. The lead asks a reviewer from another model family to review the diff and gets one finding about a daylight-saving edge case.]({{ '/assets/images/08-delegation/model-routing.png' | relative_url }}){: style="width:100%"}

*Figure 1. One task split across four tiers. Built with archify.*

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The tiers

### A common split

<!-- IMAGE NEEDED: a chart of language-model capability against cost per token, showing small, mid and large tiers; search: llm cost vs capability chart, model price performance frontier -->

This is the split my harness uses. A team with one model, or a tight budget, makes different choices.

| Tier | Use it for | Why | Typical role |
|---|---|---|---|
| Strongest | Design decisions, plans, synthesis of many reports | Mistakes here are expensive and spread into everything built on them | Lead, architect, auditor |
| Mid | Implementing a clear brief | Good code at lower cost when the decision is already made | Engineer |
| Cheapest | Bulk reading, searching, classifying | Volume work whose output is easy to check | Searcher, analyst |
| Different family | Reviewing another model's work | An independent reader catches what the author's family misses | Reviewer |

### Choosing by the cost of a mistake

One way to choose a tier is by how far a mistake would spread. A wrong decision in a plan affects every task built on it. A wrong row in a search table costs the lead one check. Under this approach a small decision still goes to the strongest tier, and a large search does not.

> Note: in my own harness, design decisions stay on the strongest tier even when they look small. A cheaper model that reaches a design question in the middle of a brief hands it back to the lead.

## Setting the model

### Per session

| Command | Sets | Applies to |
|---|---|---|
| `/model` | The model | The current session |
| `/effort` | The reasoning effort | The current session |

A sub-agent started from the session uses its profile's `model` and `effort` when they are set.

### Per role

A profile's `model` field fixes the model for every launch of that role: an alias such as `opus`, `sonnet` or `haiku`, or `inherit` to take the lead's.

```markdown
---
name: searcher
description: Finds every use of a symbol and returns a table. Read-only.
tools: Read, Grep, Glob
model: haiku
---
Return a Markdown table: file, line, one-line context. No commentary.
```

### For every sub-agent at once

| Variable | Effect |
|---|---|
| `CLAUDE_CODE_SUBAGENT_MODEL` | A default model for sub-agents whose profile names none |
| `CLAUDE_CODE_SUBAGENT_MODEL_FORCE=1` | Every sub-agent uses `CLAUDE_CODE_SUBAGENT_MODEL`, ignoring profile `model` fields (v2.1.257 and later) |

The force setting suits a session that must stay on one model, for example under a cost cap or an organization's model allowlist.

### Effort is separate from tier

Effort and tier answer different questions. Tier is how capable the model is; effort is how long it thinks on this task. A mid-tier engineer on a tricky concurrency bug may want high effort; the strongest model summarising three short reports needs little.

| Task | Tier | Effort |
|---|---|---|
| Choose between two storage designs | Strongest | High |
| Summarise three research reports already checked | Strongest | Low |
| Fix a race in a retry loop | Mid | High |
| Rename a function across forty files | Cheapest or mid | Low |

## Worked example: four tiers on one task

### The task

Make every API date carry an explicit timezone.

- The codebase: one service, a `parse_date` helper called from several modules.
- The decision to make: how dates are stored and rendered.
- The work after the decision: change the callers that pass no timezone, with tests.

### The split

1. The lead, on the strongest tier, asks a searcher on the cheapest tier for every caller of `parse_date`.
2. The searcher returns a nine-row table. The lead spot-checks two rows against the code.
3. The lead decides: store UTC, render with an offset. It writes an engineer brief for the three callers that pass no timezone.
4. The engineer, on the mid tier, writes the tests, fixes the callers and reports a commit with its test line.
5. A reviewer from a different family reads the diff and finds one issue: a report generated at a daylight-saving change.
6. The engineer fixes it; the lead checks the commit and merges.

### What it cost, in shape

| Step | Tier | Share of the reading | Share of the judgment |
|---|---|---|---|
| Search | Cheapest | Most | None |
| Decide and brief | Strongest | Little | Most |
| Build | Mid | Some | Little |
| Review | Different family | Some | Some |

## Cross-family review

### Why another family

- Models from one family share training and habits, so they tend to miss the same things.
- A reviewer from a different family reads the code without those habits.
- In practice: Claude writes and Codex reviews, or Codex writes and Claude reviews.
- In the worked example above, the cross-family reviewer found the daylight-saving case the engineer and lead had both missed.

### How the lead treats it

How the lead treats a cross-family review in my harness:

| Aspect | Practice |
|---|---|
| Role of the reviewer | An extra reader of the diff, alongside tests and the lead's own check |
| Its findings | Weighed by the lead like any other review, each fixed or answered |
| Effect on merge | It never blocks a merge on its own |
| Record | The commit or PR records which family reviewed |

See [Code review]({{ '/sections/13-code-review/' | relative_url }}).

## When to break the default

<details markdown="1">
<summary>Cases where the cheap tier is the wrong choice</summary>

- The output cannot be checked cheaply. A summary of fifty pages that nobody will re-read needs the stronger model, because nobody will catch its mistakes.
- The task hides a decision. "Rename the config keys" is mechanical until two keys collide.
- The cheaper model keeps failing. Three failed attempts at the cheap tier cost more than one at the mid tier.

</details>

<details markdown="1">
<summary>Cases where the strongest tier is wasted</summary>

- Mechanical edits with a test that proves them.
- Searches whose results the lead will check anyway.
- Formatting, renaming and moving files.

</details>

## Related pages

- [Agent profiles]({{ '/sections/08-delegation/agent-profiles.html' | relative_url }}), where the `model` field lives.
- [Checking delegated work]({{ '/sections/08-delegation/checking-delegated-work.html' | relative_url }}), what makes cheap output safe to use.

## Sources

- `/model` and `/effort`: [Claude Code docs: commands reference](https://code.claude.com/docs/en/commands.md), verified 2026-11-09.
- Profile `model` field and `inherit`: [Claude Code docs: sub-agents](https://code.claude.com/docs/en/sub-agents.md), verified 2026-11-09.
- `CLAUDE_CODE_SUBAGENT_MODEL` and `CLAUDE_CODE_SUBAGENT_MODEL_FORCE`: [Claude Code docs: Orchestrate teams of Claude Code sessions](https://code.claude.com/docs/en/agent-teams.md) (model selection), verified 2026-11-09; the force variable's effect on ordinary sub-agents is taken from that page's link to the sub-agents docs and not checked there.
- The tier split and cross-family review practice are from the author's own harness, described conceptually. The worked example is an illustration written for this page.

{% include page_nav.html %}
