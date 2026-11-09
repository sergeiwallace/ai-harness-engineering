---
title: "Trends and model changes"
nav_order: 28
has_children: true
has_toc: false
permalink: /sections/18-trends/
---

{% include page_nav.html %}

# Trends and model changes

As of 2026-11-09. Overview page. Slides for this part of the talk: [Stop 13 on the talk page]({{ '/talk/' | relative_url }}#stop-13-1-minute).

A harness is tuned to a model, and models change every few months. The practice around harnesses is also changing quickly: it has names now, the volume of agent work at frontier teams has grown by an order of magnitude, and setups have started to move between tools. This overview covers what is changing and what to do about each change. Two topic pages go deeper: [Re-testing after a model change]({{ '/sections/18-trends/retesting-after-a-model-change.html' | relative_url }}) and [Moving a harness between tools]({{ '/sections/18-trends/harness-portability.html' | relative_url }}).

![Sequence diagram of re-testing a harness after a model change. You ask the agent to compare the old and new model on the same task set; it runs each task several times on both, reads the outputs and transcripts, checks which rules never fire now and which new failures appear, proposes dropping one rule and adding one, and after your approval edits the rules and notes the model and date.]({{ '/assets/images/18-trends/model-change-retest-sequence.svg' | relative_url }}){: style="width:100%"}

*Figure 1. What to do when the model changes: run the same tasks on both, read the differences, change the harness and record why. Diagram made with archify.*

| Trend | What it means for your harness | Where it goes deeper |
|---|---|---|
| New model generations | Re-test rules; remove the ones that no longer help | [Re-testing after a model change]({{ '/sections/18-trends/retesting-after-a-model-change.html' | relative_url }}) |
| Named practices | Shared vocabulary and published patterns to learn from | [AI harnesses]({{ '/sections/02-ai-harnesses/' | relative_url }}) |
| Agent work at scale | The system around the agents decides what lands | [Orchestration]({{ '/sections/10-orchestration/' | relative_url }}) |
| Portable setups | Skills and context files can move between tools | [Moving a harness between tools]({{ '/sections/18-trends/harness-portability.html' | relative_url }}) |
| Parallel agents in the client | Less need to build your own coordination | [Coordinating several sessions]({{ '/sections/09-session-coordination/' | relative_url }}) |

## 1. New models change what a harness needs

### 1.1 Rules written for an older model

Many rules in a harness exist to compensate for something a model did badly. When the next model stops doing it, the rule only adds cost: it takes context on every session, and it can push the new model away from behaviour that is now better than what the rule asks for. Instructions written for an older model can waste tokens or make results worse with a newer one.

### 1.2 New habits need new rules

The reverse also happens. A new model can bring habits the old one did not have, and your rules have to catch up.

#### Worked example: writing tells

My writing rules list the patterns that make text read as machine-written. When the model changed, the list changed with it:

| Pattern | Older model | Newer model | What I did |
|---|---|---|---|
| Em dashes | Frequent | Nearly gone | Kept a short rule; it rarely fires now |
| Bold lead-in labels on paragraphs | Occasional | More frequent | Added a rule |
| Three-part lists ("fast, cheap and reliable") | Occasional | More frequent | Added a rule |
| Hedges ("this may help in some cases") | Occasional | More frequent | Added a rule |

I found these by reading the new model's output against the old rules. The release notes did not mention them.

### 1.3 Re-test when the model changes

Treat a model change like any other change to the harness: run the same tasks on both and compare. The [re-testing page]({{ '/sections/18-trends/retesting-after-a-model-change.html' | relative_url }}) has the procedure, a task-set format and a worked comparison. This is the same pruning step as in [complexity creep]({{ '/sections/17-building-a-harness/complexity-creep.html' | relative_url }}).

> Note: re-test on a model change you did not choose, too. A client can change its default model, and the harness is then running against a new model without anyone deciding it.

## 2. Harness engineering has names now

### 2.1 Context engineering

Anthropic described context engineering in September 2025 as the work of choosing what goes into the model's limited context at each step: instructions, tools, examples, history and retrieved data. The framing moved attention from writing one good prompt to managing what the agent sees over a long task. See [The context window]({{ '/sections/11-context-window/' | relative_url }}).

### 2.2 Harness engineering

Through 2026, essays and reports used "harness engineering" for the whole system around the model: context files, tools, hooks, checks, sub-agents and the human gates. This site uses the term in that sense; [AI harnesses]({{ '/sections/02-ai-harnesses/' | relative_url }}) defines it.

## 3. Agent work at scale

Merge volume at frontier teams has grown to the point that the system around the agents decides most of what lands. Anthropic reported:

> "In May 2026, Anthropic averaged around 2,500 code changes merged per day; as of September, we merged over 30,000 per day."

It added: "The bottleneck, now, is all of the developer infrastructure built for human scale: source control, builds, and CI."

### What this means at a smaller scale

At one person's scale the numbers are smaller, and the bottleneck moves in the same way. Once agents produce work faster than you can review it, the bottleneck moves to the checks, the merge path and your own attention. The parts of a harness that matter most become the [tests]({{ '/sections/12-testing/' | relative_url }}), the [review]({{ '/sections/13-code-review/' | relative_url }}) and the [gates]({{ '/sections/14-human-gates/' | relative_url }}).

## 4. Setups move between tools

The Agent Skills format is shared, so a skill written for Claude Code can be read by other tools that support the standard. Codex can import a Claude Code setup. Context files have converged too: Claude Code reads `CLAUDE.md`, Codex reads `AGENTS.md`, and one can point at the other. The [portability page]({{ '/sections/18-trends/harness-portability.html' | relative_url }}) covers what moves cleanly and what does not.

## 5. Parallel agents inside the client

Claude Code now offers several ways to run agents in parallel, some still in preview, such as agent view and Projects. Work that once needed a hand-built coordinator, such as several sessions on one repository, increasingly comes with the client. Before building coordination yourself, check what the client already does: see [Coordinating several sessions]({{ '/sections/09-session-coordination/' | relative_url }}).

## Examples from my own harness

The writing-tells table above is the clearest case: a model change made some of my rules idle and showed patterns no rule covered. I now treat a model change as a reason to read a batch of fresh output against my rules, and to delete as well as add.

To start: the next time your model changes, run five tasks you know well on the old and the new model, and compare the outputs side by side.

## Pitfalls and lessons

<details markdown="1">
<summary>Assuming a newer model needs the same harness</summary>

Some rules written for an older model make a newer one worse. For example, a rule that forces step-by-step planning could slow a model that already plans well on its own. Re-test before assuming.

</details>

<details markdown="1">
<summary>Building what the client is about to ship</summary>

Clients add features quickly. A coordination layer, a memory system or a review command you build this month may arrive natively next month. Prefer small mechanisms you can retire over large ones you have to maintain.

</details>

## Pages in this section

- [Re-testing after a model change]({{ '/sections/18-trends/retesting-after-a-model-change.html' | relative_url }}), with the sub-topic [Building a task set]({{ '/sections/18-trends/building-a-task-set.html' | relative_url }})
- [Moving a harness between tools]({{ '/sections/18-trends/harness-portability.html' | relative_url }})

## Sources

- Context engineering: <https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents>, verified 2026-11-08.
- Parallel agents in Claude Code: <https://code.claude.com/docs/en/agents.md>, verified 2026-11-08.
- Codex `/import`: <https://learn.chatgpt.com/docs/developer-commands.md?surface=cli>, verified 2026-11-08.
- Anthropic merged-changes figures and the bottleneck quote: <https://www.anthropic.com/research/redesigning-source-control>, verified 2026-11-09.

{% include page_nav.html %}
