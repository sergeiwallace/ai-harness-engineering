---
title: "Detail"
parent: "Orchestration and workflows"
nav_order: 1
---

{% include page_nav.html %}

# Orchestration in detail

As of 2026-11-09.

## Purpose

Cover long-running and multi-step work: the two kinds of orchestration, goals and loops.

## Key points

- Prompt-driven orchestration runs from prompts and agent profiles. A coordinator session or lead agent writes briefs, launches sub-agents and joins their results. It fits work whose steps depend on judgment and change as the work goes.
- Programmatic orchestration holds the plan in code. It fits work with known steps, many parallel agents, or a need to resume after a failure.
  - Claude Code's dynamic workflows are JavaScript scripts that run many sub-agents. The plan does not take up Claude's context. Runs are watched with `/workflows`. By default a run allows 16 agents at once and 1,000 in total.
  - LangGraph describes a workflow as a graph of steps with shared state, retries and human approval points. Other frameworks in use include the OpenAI Agents SDK, CrewAI, Google's Agent Development Kit, and Microsoft Agent Framework, which replaced the archived AutoGen.
- `/goal` adds a check at the end of each turn. A small fast model reads the conversation, without calling tools, and decides whether the condition is met. Write the condition so it names evidence the agent will print. Anthropic's guidance for Opus 5.5 recommends the same pattern for custom harnesses, and stopping after two or three automatic continuations.
- `/loop` repeats a prompt on a time interval, for monitoring or recurring work.
- Every loop needs an exit condition and a limit.
- Hill-climbing: run, measure, change one thing, and keep the change only if the measure improved.
- `/workflows`: Watches, pauses and saves dynamic workflows, the scripts Claude writes to run many sub-agents. A workflow starts when you ask for one or use the `ultracode` keyword.

## Examples from my own harness

- Goal templates, such as "work every task in progress to the end", started and ended through skills.
- A prompt-driven pipeline that runs research, decision, audit, build and a second audit, with coordinator agents doing each step.
- A quality ratchet that lets code-quality numbers improve and refuses a change that makes them worse.

To start: one `/goal` that has a clear finish condition.

## Pitfalls and lessons

- Adding a fix for each failure grows an orchestration engine until the fixes cause failures. Cutting steps can work better than adding them.

## Sources

- Dynamic workflows: <https://code.claude.com/docs/en/workflows.md>, verified 2026-11-08.
- LangGraph: <https://docs.langchain.com/oss/python/langgraph/overview>, verified 2026-11-08.
- AutoGen archived: <https://github.com/microsoft/autogen>, verified 2026-11-08.
- `/goal`: <https://code.claude.com/docs/en/goal.md>, verified 2026-11-08.
- Opus 5.5 harness guidance: <https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5-5>, verified 2026-11-08.
- `/loop`: <https://code.claude.com/docs/en/scheduled-tasks.md>, verified 2026-11-08.

{% include page_nav.html %}
