---
title: "Trends and model changes"
nav_order: 28
has_children: true
has_toc: false
permalink: /sections/18-trends/
last_updated: "2026-11-09"
talk_stop: "13"
---

{% include page_nav.html %}

# Trends and model changes

A harness is tuned to a model, and models change every few months. The practice around harnesses is also changing quickly: it has names now, Anthropic reports merging about twelve times as many code changes per day in September 2026 as in May, and setups have started to move between tools. This overview covers what is changing and what to do about each change. Two topic pages go deeper: [Re-testing after a model change]({{ '/sections/18-trends/retesting-after-a-model-change.html' | relative_url }}) and [Moving a harness between tools]({{ '/sections/18-trends/harness-portability.html' | relative_url }}).

![Sequence diagram of re-testing a harness after a model change. You ask the agent to compare the old and new model on the same task set; it runs each task several times on both, reads the outputs and transcripts, checks which rules never fire now and which new failures appear, proposes dropping one rule and adding one, and after your approval edits the rules and notes the model and date.]({{ '/assets/images/18-trends/model-change-retest-sequence.svg' | relative_url }}){: style="width:100%"}

*Figure 1. What to do when the model changes: run the same tasks on both, read the differences, change the harness and record why. Diagram made with archify.*

| Trend | What it means for your harness | Where it goes deeper |
|---|---|---|
| New model generations | Re-test rules; remove the ones that no longer help | [Re-testing after a model change]({{ '/sections/18-trends/retesting-after-a-model-change.html' | relative_url }}) |
| Named practices | Shared vocabulary and published patterns to learn from | [What is a harness?]({{ '/sections/02-what-is-a-harness/' | relative_url }}) |
| Agent work at scale | The system around the agents decides what lands | [Orchestration]({{ '/sections/10-orchestration/' | relative_url }}) |
| Portable setups | Skills and context files can move between tools | [Moving a harness between tools]({{ '/sections/18-trends/harness-portability.html' | relative_url }}) |
| Parallel agents in the client | Less need to build your own coordination | [Coordinating several sessions]({{ '/sections/09-session-coordination/' | relative_url }}) |

## 1. New models change what a harness needs

### 1.1 Rules written for an older model

Many rules in a harness exist to compensate for something a model did badly. When the next model stops doing it, the rule only adds cost:

- It takes context on every session.
  - A layer 1 rule is read at the start of every session, whether or not it still applies.
- It can push the new model away from better behaviour.
  - For example, a rule that forces a fixed answer structure can override a model that now structures answers well for each case.
- It adds a check that never fires.
  - A hook for a failure that no longer happens still runs on every matching event.

Instructions written for an older model can waste tokens or make results worse with a newer one.

### 1.2 New habits need new checks

The reverse also happens. A new model can bring habits the old one did not have, and the harness has to catch up:

- With a check, where a program can detect the habit.
- With a short rule, where it cannot.

#### Worked example: writing tells

My writing rules list the patterns that make text read as machine-written. When the model changed, the list changed with it:

| Pattern | Older model | Newer model | What I did |
|---|---|---|---|
| Em dashes | Frequent | Nearly gone | Kept a short rule and the pre-publish grep; both rarely fire now |
| Bold lead-in labels on paragraphs | Occasional | More frequent | Added to the list; a pattern check can detect it |
| Three-part lists ("fast, cheap and reliable") | Occasional | More frequent | Added to the list; needs a reader's judgment |
| Hedges ("this may help in some cases") | Occasional | More frequent | Added to the list; a pattern check can catch the common words |

I found these by reading the new model's output against the old rules. The release notes did not mention them.

### 1.3 Re-test when the model changes

Treat a model change like any other change to the harness:

1. Run the same [task set]({{ '/sections/18-trends/building-a-task-set.html' | relative_url }}) on the old and the new model, several times each.
2. Read the differences: idle rules, rules that now hurt, and new failures.
3. Change one thing at a time, and put each new fix in the layer that can enforce it.
4. Record the model, the date and the reason for each change.

The [re-testing page]({{ '/sections/18-trends/retesting-after-a-model-change.html' | relative_url }}) has the full procedure and a worked comparison. This is the same pruning step as in [complexity creep]({{ '/sections/17-building-a-harness/complexity-creep.html' | relative_url }}).

### 1.4 Which layers a model change affects

The [four layers]({{ '/sections/17-building-a-harness/' | relative_url }}#11-the-four-layers) differ in how much a new model changes them.

| Layer | Effect of a model change | What to re-check |
|---|---|---|
| 1. Instructions | The most affected: rules written for the old model's habits may now be idle or harmful, and new habits are uncovered | Every always-loaded rule against fresh output |
| 2. Workflows and tools | Mostly unaffected: the steps still run the same way | Steps that worked around an old weakness, such as splitting a task the new model can do in one pass |
| 3. Hooks, scripts and linters | Unaffected in how they run; how often they fire can change | Fire counts and false blocks in the first weeks |
| 4. Validation and tests | Unaffected | Nothing; they are the yardstick for the re-test |

A harness whose enforcement sits in layers 2 to 4 needs less rework when the model changes, because the parts that do the enforcing do not depend on the model.

## 2. Harness engineering has names now

### 2.1 Context engineering

Anthropic described context engineering in September 2025 as the work of choosing what goes into the model's limited context at each step. Its parts:

- System instructions and context files.
- Tool definitions.
- Examples.
- Message history.
- Data retrieved during the task.

The framing moved attention from writing one good prompt to managing what the agent sees over a long task. See [The context window]({{ '/sections/11-context-window/' | relative_url }}).

### 2.2 Harness engineering

Through 2026, essays and reports used "harness engineering" for the whole system around the model.

| Term | Scope |
|---|---|
| Prompt engineering | One prompt |
| Context engineering | What the model sees at each step of a task |
| Harness engineering | The whole system: context, tools and workflows, hooks and checks, sub-agents, and the human gates |

This site uses the term in that sense, with a systems approach to building it; [What is a harness?]({{ '/sections/02-what-is-a-harness/' | relative_url }}) defines it.

## 3. Agent work at scale

Merge volume at frontier teams has grown to the point that the system around the agents decides most of what lands. Anthropic reported:

> "In May 2026, Anthropic averaged around 2,500 code changes merged per day; as of September, we merged over 30,000 per day."

It added: "The bottleneck, now, is all of the developer infrastructure built for human scale: source control, builds, and CI."

### What this means at a smaller scale

At one person's scale the numbers are smaller, and the bottleneck moves in the same way. Once agents produce work faster than you can review it, it moves to:

1. The checks.
   - [Tests]({{ '/sections/12-testing/' | relative_url }}) and validation decide what is fit to merge before you look at it.
2. The merge path.
   - A slow test run or a queue of pull requests holds up every agent.
3. Your own attention.
   - [Review]({{ '/sections/13-code-review/' | relative_url }}) and [human gates]({{ '/sections/14-human-gates/' | relative_url }}) should reach you only for what needs a person.

## 4. Setups move between tools

| Part | What changed |
|---|---|
| Skills | The Agent Skills format is shared, so a skill written for Claude Code can be read by other tools that support the standard |
| Whole setups | Codex can import a Claude Code setup with `/import` |
| Context files | Claude Code reads `CLAUDE.md`, Codex reads `AGENTS.md`, and one can point at the other |
| Hooks and approvals | Still tool-specific; each needs rewriting for the target |

The [portability page]({{ '/sections/18-trends/harness-portability.html' | relative_url }}) covers what moves cleanly and what does not.

## 5. Parallel agents inside the client

Claude Code now offers several ways to run agents in parallel, some still in preview, such as agent view and Projects. Work that once needed a hand-built coordinator increasingly comes with the client.

1. Check what the client already does before building coordination yourself.
   - Several sessions on one repository, sub-agents, background tasks.
2. Build only the gap.
   - For example, a shared record of who owns which task, if the client has none.
3. Re-check when the client updates, and retire what it now covers.

See [Coordinating several sessions]({{ '/sections/09-session-coordination/' | relative_url }}).

## Examples from my own harness

- The writing-tells table above is the clearest case.
  - A model change made some of my rules idle and showed patterns no rule covered.
- I now treat a model change as a reason to re-check the harness.
  - Read a batch of fresh output against the rules.
  - Delete idle rules as well as adding new ones.
  - Move patterns a program can detect into a pattern check, so they do not depend on the model reading the list.

To start: the next time your model changes, run five tasks you know well on the old and the new model, and compare the outputs side by side.

## Pages in this section

- [Re-testing after a model change]({{ '/sections/18-trends/retesting-after-a-model-change.html' | relative_url }}), with the sub-topic [Building a task set]({{ '/sections/18-trends/building-a-task-set.html' | relative_url }})
- [Moving a harness between tools]({{ '/sections/18-trends/harness-portability.html' | relative_url }})

## Sources

- Context engineering: [Anthropic, Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents), verified 2026-11-08.
- Parallel agents in Claude Code: [Claude Code docs: Agents](https://code.claude.com/docs/en/agents.md), verified 2026-11-08.
- Codex `/import`: [Codex docs: Developer commands](https://learn.chatgpt.com/docs/developer-commands.md?surface=cli), verified 2026-11-08.
- Anthropic merged-changes figures and the bottleneck quote: [Anthropic, Redesigning source control](https://www.anthropic.com/research/redesigning-source-control), verified 2026-11-09.

{% include page_nav.html %}
