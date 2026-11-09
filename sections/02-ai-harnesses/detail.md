---
title: "Detail"
parent: "AI harnesses"
nav_order: 1
---

# AI harnesses in detail

As of 2026-11-09.

## Purpose

Define a harness and its parts, so later pages share one vocabulary.

## Key points

- A harness is the software around a model that gives it safe access to tools, memory and checks, set up to match one person's preferences, workflows and projects. This is the definition from the meeting description.
- A 2026 definition from practice: "harness has emerged as a shorthand to mean everything in an AI agent except the model itself" (Birgitta Böckeler, Thoughtworks).
- Its parts, each with its own page:
  - Context: instructions and knowledge the model reads (see [Context files]({{ '/sections/04-context-files/' | relative_url }}) and [Templates, prompt stubs and injected prompts]({{ '/sections/07-templates-and-stubs/' | relative_url }})).
  - Capabilities: skills, commands, plugins and tools (see [Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }})).
  - Controls: hooks, scripts and checks that run whether or not the model remembers a rule (see [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }})).
  - Workers: agent profiles and sub-agents (see [Agent profiles, sub-agents and delegation]({{ '/sections/08-delegation/' | relative_url }})).
  - Coordination: how sessions and agents work together (see [Coordinating several sessions]({{ '/sections/09-session-coordination/' | relative_url }}) and [Orchestration and workflows]({{ '/sections/10-orchestration/' | relative_url }})).
  - Memory: what carries over between sessions (see [Context window and memory]({{ '/sections/11-context-window/' | relative_url }})).
  - Verification: tests, review, human gates and browser checks (see [Testing agent-written code]({{ '/sections/12-testing/' | relative_url }}), [Code review]({{ '/sections/13-code-review/' | relative_url }}), [Human gates, decisions and alignment]({{ '/sections/14-human-gates/' | relative_url }}) and [Browser automation and computer use]({{ '/sections/15-browser-automation/' | relative_url }})).
  - Measurement: logs and evaluation (see [Logging, observability and evaluation]({{ '/sections/16-logging-and-evaluation/' | relative_url }})).
- Böckeler sorts the parts into guides, which steer the agent before it acts, and sensors, which give it feedback after it acts.
- Some parts are instructions the model interprets. Others are code that always runs the same way. Claude Code's docs call `CLAUDE.md` "context, not enforced configuration". [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) covers when to move a rule from the first kind to the second.
- Research that measured harnesses found they changed task success by a few points and changed cost a lot (not yet verified against the primary source).

## Examples from my own harness

- A ladder for each rule, from soft to hard: a context rule, a path rule, a skill, a hook or guard, and a test or CI gate (see [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }})). A rule that agents keep breaking moves up to the next step that can enforce it. Actions only a person may take sit beside the ladder as human approvals (see [Human gates, decisions and alignment]({{ '/sections/14-human-gates/' | relative_url }})).

To start: use the same ladder for your own rules.

## Pitfalls and lessons

- An instruction in a context file is easy to add and is often ignored.
- Every part that loads costs tokens on every turn. Measure before adding.

## Sources

- Böckeler, "Harness engineering": <https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html> (2026-04-02), verified 2026-11-08.
- "context, not enforced configuration": <https://code.claude.com/docs/en/memory.md>, verified 2026-11-08.
- Harness cost measurements (2 to 5 points of success across 21 model and harness pairs, and Claude Code at about twice the cost of a minimal four-tool harness on SWE-bench Lite): reported by HarnessTax (UC Berkeley and Arena, 2026), <https://harnesstax.github.io/>; not yet verified against the primary source.
