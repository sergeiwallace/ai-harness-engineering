---
title: "Chaining skills into a pipeline"
parent: "Prompt-driven orchestration"
grand_parent: "Orchestration and workflows"
nav_order: 1
---

{% include page_nav.html %}

# Chaining skills into a pipeline

As of 2026-11-09.

A sub-topic of [prompt-driven orchestration]({{ '/sections/10-orchestration/prompt-driven-orchestration.html' | relative_url }}). A pipeline is a fixed order of stages, each run by a [skill]({{ '/sections/05-skills-and-plugins/' | relative_url }}) and usually by its own agent, where each stage's output is a file the next stage reads.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The stages

![A sequence diagram. The research agent writes research to files. The decide agent reads it and writes a plan. Inside a loop fragment the audit agent reads the plan and sends must-fix findings to the decide agent, which revises the plan, until the audit finds nothing new. The build agent then reads the plan and writes code.]({{ '/assets/images/10-orchestration/pipeline-stages.svg' | relative_url }}){: style="width:100%"}

*Figure: stages hand files to each other; the audit loop ends when nothing new turns up. Made with archify.*

| Stage | Reads | Writes | Ends when |
|---|---|---|---|
| Research | The request | A research doc with sources | Every question has an answer or is marked open |
| Decide | Research doc | A plan doc with each decision and its reason | No decision is left open |
| Audit the plan | Plan doc | Findings, sorted by severity | A round finds nothing new to fix |
| Build | Plan doc | Code, tests, docs | The plan's acceptance criteria pass |
| Audit the build | Plan and diff | Findings | A round finds nothing new to fix |

### Files between stages

Each stage hands over a file. That has three effects:

| Effect | Example |
|---|---|
| A stage runs in a fresh agent with a clean context | The build agent reads only the plan doc |
| A person can read and correct the file between stages | You edit one decision in the plan before the build starts |
| A failed stage reruns from the last good file | The build fails; it restarts from the same plan |

### Audits with exits

Both audit stages loop back on must-fix findings, so each needs an exit. Two exit rules I use together:

- Stop when a round finds nothing new.
- Cap the rounds, for example at three.

[Code review]({{ '/sections/13-code-review/' | relative_url }}) covers why an open-ended "find problems" audit does not converge on its own.

## Worked example: adding rate limiting

#### Research

The research agent reads the service's request path and three rate-limiting approaches, and writes a doc: token bucket per client fits, the existing cache can hold the counters, two endpoints are exempt by contract.

#### Decide

The decision agent writes the plan: token bucket, limits in a config file, exempt endpoints listed, acceptance criteria as tests ("the 101st request in a minute gets 429").

#### Audit the plan

Round one finds that the plan does not say what happens when the cache is down. That is a must-fix; the plan gains "fail open, log a warning". Round two finds nothing new. The stage ends.

#### Build and audit

The build agent writes the code and the tests from the acceptance criteria. The build audit flags a missing test for the exempt endpoints; the build agent adds it; the next round is clean.

> Each stage's agent started fresh and read only the previous stage's file. None of them needed the whole history in context.
{: .note }

## Designing your own

1. Name each stage's input file and output file.
2. Give each stage an end condition a reader can check.
3. Put a person between stages where a decision is theirs to make ([human gates]({{ '/sections/14-human-gates/' | relative_url }})).
4. Cap every loop.

<details markdown="1">
<summary>When a pipeline is too much</summary>

A two-line fix does not need research, a plan doc and two audits. Pipelines pay off on work that spans days, involves decisions a person will want to read, or will be handed between sessions. For small work, one agent with a clear `/goal` is enough.

</details>

## Examples from my own harness

- A prompt-driven pipeline that runs research, decision, audit, build and a second audit, with coordinator agents doing each step.
  - Its state lives in a file, so a stage that stops can be resumed.
  - Each audit loop has a round cap and a "nothing new" stop.

To start: two stages, plan then build, with the plan as a file you read before the build starts.

{% include page_nav.html %}
