---
title: "Guides, sensors and enforcement"
parent: "What is a harness?"
nav_order: 8
redirect_from:
  - /sections/02-ai-harnesses/guides-sensors-and-enforcement.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Guides, sensors and enforcement

The components of a harness can be sorted two ways, and the rest of this site uses both. By timing: guides act before the agent does anything, sensors after. By strength: soft parts are text the model interprets, hard parts are code that runs every time. This page covers both, the enforcement ladder that connects them, scripts and templates as enforcement, what a harness costs, and how to read a harness you did not build.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Guides and sensors

### The two kinds

Birgitta Böckeler of Thoughtworks sorts harness parts into guides and sensors.

| | Guides | Sensors |
|---|---|---|
| When they act | before the agent acts | after the agent acts |
| What they do | steer the next action | report on the last action |
| Examples | context files, skills, templates, agent profiles | tests, linters, type checks, review, hooks that inspect output |
| Typical failure | ignored, or crowded out of the context | missing, slow, or easy to bypass |
| Cost | tokens on every turn | time on every check |

### Why a harness needs both

| With only | What happens |
|---|---|
| guides | the agent usually does the right thing; when it does not, nothing notices |
| sensors | every mistake is caught, but only after the time to make it is spent |
| both | mistakes are rarer, and the rare ones are caught before they ship |

### A worked example

A database migration.

1. Guide: a skill for schema changes tells the agent to write a reversible migration with a down step.
2. Sensor: a test applies and reverts every migration on a scratch database.
3. Result: the guide makes a missing down step rare; the sensor catches the one that slips through, before it ships.

## Soft and hard parts

### Instructions the model interprets

A context-file line, a skill or a template is text the model reads and usually follows. "Usually" weakens:

- as the context window fills;
- when the agent juggles several tasks;
- when it switches to something else and forgets to come back.

> Claude Code's docs describe `CLAUDE.md` as "context, not enforced configuration". They add two pieces of guidance: "The more specific and concise your instructions, the more consistently Claude follows them", and, to block an action regardless of what Claude decides, use a `PreToolUse` hook.
{: .note }

### Code that always runs

A hook, script, linter, test or CI gate runs the same way every time. It cannot be forgotten or argued with. It can be wrong, slow or too strict, so it needs maintenance like any code; [Hook failure modes]({{ '/sections/02-what-is-a-harness/hook-failure-modes.html' | relative_url }}) lists the ways.

| | Soft | Hard |
|---|---|---|
| Cost to write | minutes | hours |
| Followed | most of the time | every time it runs |
| Good for | judgment calls, preferences, style | rules that must always hold |
| Fails by | being forgotten or skimmed | being wrong, slow, bypassed, or never installed |

## The enforcement ladder

![A vertical flow chart. One agent action enters at the top and moves down through four layers. Layer 1, instructions in context files, skills and prompts, is read by the model and guides it. Layers 2 to 4 are enforced by a program: workflows and tools refuse invalid input, hooks, scripts and linters refuse or flag the action as it happens, and validation and tests fail the change and send it back. At each layer a side branch shows what that layer caught, and what it missed passes down: a rule the model dropped, an action outside the workflow, a mistake not visible yet. A change that passes all four layers lands.]({{ '/assets/images/diagrams/02-ai-harnesses/enforcement-flow.svg' | relative_url }})
*Figure: one agent action moving down the four layers. Each layer catches some mistakes and passes the rest to the next. Made with archify.*

*Placeholder image: the Swiss cheese model of layered defence (James Reason). Each layer catches some mistakes; what gets through one layer is caught by the next. Source: [Swiss cheese model of accident causation on Wikimedia Commons](https://commons.wikimedia.org/wiki/File:Swiss_cheese_model_of_accident_causation.png), by Davidmack, [CC BY-SA 3.0](https://creativecommons.org/licenses/by-sa/3.0/). Resized for this page.*

### The layers

| Layer | What it is | Enforced by | Example from my harness | What it misses, for the next layer to catch |
|---|---|---|---|---|
| 1. Instructions | context files, path rules, skill text, prompts | the model, reading it | "never kill processes by name pattern" in the context file | a model juggling tasks or near a full window drops the line |
| 2. Workflows and tools | skills with fixed steps, scripts, templates, tools that accept only valid input | the program the agent runs | a commit script that builds the message and refuses one without the delegation line | actions the agent takes outside the workflow, such as a raw shell command |
| 3. Hooks, scripts and linters | checks at the moment of action | the harness, every time | a `PreToolUse` guard that refuses `pkill` and names the safe form | spellings the guard does not parse, and mistakes only visible in the finished result |
| 4. Validation and tests | tests, CI, link checkers, review before a change lands | programs and reviewers on the result | a link checker that walks the built site before publish | design and judgment problems no check encodes; those go to a person |

### The steps in Claude Code

The same ladder, as the concrete places a rule can live:

| Step | Layer | Example | Checked when |
|---|---|---|---|
| context rule | 1 | "Run the tests before committing." | if the model remembers |
| path rule | 1 | test standards in `.claude/rules/` for test files | when the agent opens a matching file |
| skill or script | 2 | a commit procedure with a test step, or a commit script that runs the tests | when the workflow runs |
| hook or guard | 3 | a `PreToolUse` hook that refuses the commit command until tests pass | as the agent acts |
| test or CI gate | 4 | a git pre-push hook and a CI job that run the suite | at push and merge |

These are the layers of the [systems approach]({{ '/sections/02-what-is-a-harness/' | relative_url }}#a-systems-approach-to-harness-engineering). My harness enforces each rule in the latest layer that can check it, and keeps a context-file line only as a second layer. Actions only a person may take, such as publishing or deleting, sit beside the ladder as human approvals; see [Human gates]({{ '/sections/14-human-gates/' | relative_url }}).

### One rule moved into the system

The rule: never cite a file that is not committed as evidence.

1. It started as a context-file line, then as a line in the sub-agent brief template.
   - Agents kept citing files in ignored build directories anyway.
   - Adding more wording would have been the anti-pattern: more text, applied just as unevenly.
2. The fix went into the system as a hook that checks every citation path in a document being written.
   - It refuses an edit whose evidence points at an untracked file.
   - Its refusal says which path was untracked and how to cite a committed one.
3. The context-file line stays as a second layer, so the model's own behaviour matches what the hook requires.

In hindsight the rule could have gone to the hook first: the bad action is visible in the tool input, and a wrong citation is costly to find later.

<details markdown="1">
<summary>When a rule stays a guide</summary>

Some rules cannot be checked from the action alone: "prefer the simpler design", "ask when the requirement is unclear". Forcing them into a hook produces a guard that refuses good work or passes bad work. Leave them as guides and add a sensor that sees the whole result, such as a review step with the rule in its checklist.

</details>

## Scripts, linters and tests

### Scripts as enforcement

A multi-step instruction is fragile: the agent may do steps two and four and skip three. A script that does all four, and refuses to run with missing inputs, makes the procedure one command.

```text
$ commit-with-provenance --title "docs: add hooks page" --body-file body.txt
refused: the body must say whether the change was delegated, and to whom
```

My harness builds every commit message through a script like this. As a written rule, agents forgot the delegation line; as a script, it is never missing.

### Linters and type checkers

Fast sensors that run on every file the agent writes.

```json
{ "hooks": { "PostToolUse": [ { "matcher": "Edit|Write",
  "hooks": [ { "type": "command", "command": "uv run ruff check --quiet" } ] } ] } }
```

- Run from a `PostToolUse` [hook]({{ '/sections/02-what-is-a-harness/hook-lifecycle-events.html' | relative_url }}#posttooluse), so the model sees the problem while the edit is fresh.
- Lint only the file just written: a script reads `tool_input.file_path` from the hook's input and passes it to the linter, so a large repository does not slow every edit.
- Run again at commit, so nothing that skipped the hook lands.

### Tests and gates

The slowest and strongest sensors.

| Gate | Runs | Typical content |
|---|---|---|
| git pre-commit hook | on `git commit` | lint, fast tests |
| git pre-push hook | on `git push` | the full suite |
| CI | on every change | the full suite on a clean machine |

A test that pins a rule ("no live link points into a temporary worktree") keeps the rule true after everyone has forgotten why it exists. See [Testing]({{ '/sections/12-testing/' | relative_url }}).

## Templates as guides

A template fixes the shape of a brief, a plan or a report.

- As a guide, it makes the agent fill in every field.
- As a partner to a sensor, it makes the result checkable: a report with fixed fields can be checked by code for missing ones.
- My harness's sub-agent briefs carry the founding ask verbatim, the agent's scope, binding safety limits and a fixed report shape.
- A `SubagentStop` check sends a report back when a field is missing.

See [Templates, prompt stubs and injected prompts]({{ '/sections/07-templates-and-stubs/' | relative_url }}).

## Harness cost

### Tokens on every turn

Everything that loads at startup is paid for on every turn.

| Loads every turn | Typical size |
|---|---|
| context files and rules | hundreds to thousands of tokens |
| tool and MCP descriptions | tens to hundreds per tool |
| skill summaries | one line per skill |
| hook-injected reminders | per event, if the hook adds context |

A part that costs a few hundred tokens and loads everywhere costs those tokens thousands of times a day.

### What research measured

HarnessTax (UC Berkeley and Arena, 2026) compared model and harness pairs.

- Across 21 pairs, the harness moved task success by about 2 to 5 points.
- Claude Code cost about twice as much as a minimal four-tool harness on SWE-bench Lite.
- These figures are not yet verified against the primary source.

LangChain measured what a harness can gain. Its deepagents-cli went from 52.8% to 66.5% on Terminal Bench 2.0 with the model fixed at gpt-5.2-codex, by changing only the harness (LangChain, 2026-02-17). One of the changes was a check that stops the agent before it exits and asks for a verification pass, the same idea as a `Stop` hook.

> Benchmark tasks are short and self-contained. Much of what a personal harness does (memory, coordination, approvals) never shows on them, so read the figures as a warning about cost; they say little about any one harness.
{: .note }

### A worked example

A context file that grew to several hundred lines.

1. The always-needed rules stay, each in one sentence.
2. Detail moves to procedure documents, each with a one-line pointer.
3. File-specific rules move to path rules.
4. Rules a hook or test now enforces shrink to a pointer to the check.
5. `/context` in a fresh session shows the smaller standing cost.

[Context file size budgets]({{ '/sections/02-what-is-a-harness/context-file-budgets.html' | relative_url }}) has the full trim ladder.

## Reading a harness you did not build

When you adopt someone else's setup, or open your own after a few months, three questions find most problems.

1. What loads on every turn? List the context files, rules, tool descriptions and hooks that inject text. Each is a fixed cost.
2. What enforces in code? List the hooks, checks and gates. For each, find the rule it enforces and the incident it came from.
3. What has not fired in a long time? A rule nobody breaks any more, or a check that never fails, may compensate for an older model and can go.

## Pitfalls

| Pitfall | Better |
|---|---|
| Adding a context-file line for every mistake | put the fix in a workflow, hook, script or test; keep one line as a second layer |
| Adding parts without measuring | check `/context` before and after; remove what does not help |
| Mechanism on mechanism | see complexity creep in [Building your own harness]({{ '/sections/17-building-a-harness/' | relative_url }}) |

## Sources

- [Böckeler, "Harness engineering"](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html) (2026-04-02), verified 2026-11-08.
- [Claude Code memory docs](https://code.claude.com/docs/en/memory.md), source of "context, not enforced configuration" and the guidance quoted with it, verified 2026-11-09.
- [Swiss cheese model of accident causation](https://commons.wikimedia.org/wiki/File:Swiss_cheese_model_of_accident_causation.png), Wikimedia Commons, by Davidmack, [CC BY-SA 3.0](https://creativecommons.org/licenses/by-sa/3.0/). The model is James Reason's.
- Harness cost measurements: reported by [HarnessTax](https://harnesstax.github.io/) (UC Berkeley and Arena, 2026); not yet verified against the primary source.
- [LangChain, "Improving deep agents with harness engineering" (2026-02-17)](https://www.langchain.com/blog/improving-deep-agents-with-harness-engineering), verified 2026-11-09.
- The enforcement ladder, the citation example and the commit script are from the author's own harness, described conceptually.

{% include page_nav.html %}
