---
title: "Document and goal templates"
parent: "Templates, prompt stubs and injected prompts"
nav_order: 3
---

{% include page_nav.html %}

# Document and goal templates

As of 2026-11-09. Topic page in [Templates, prompt stubs and injected prompts]({{ '/sections/07-templates-and-stubs/' | relative_url }}).

A document template is the full shape of one kind of document, with a description of what goes under each heading. A goal template is the shape of a stop condition for `/goal`. Prompt stubs are read once, at launch, by one agent; these are read later, by many readers, or after every turn by an evaluator. This page covers both, with a template per common document kind and a set of checkable goals.

![A lifecycle from left to right: goal armed, agent works, turn ends, evaluator reads, goal met. From the evaluator, not yet loops back to agent works, and a stated blocker leads to a blocker-reported state. From agent works, a started job leads to an idle state that returns on an injected wake-up.]({{ '/assets/images/07-templates-and-stubs/goal-evaluator.png' | relative_url }}){: style="width:100%"}

*A goal from start to stop: the evaluator reads the conversation after every turn. Built with archify.*

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Document templates

### What they are for

An agent asked to "write up the research" produces a different shape every time. A template fixes that.

- The reader always finds the answer first and the sources last.
- A reviewer can check a document against its kind: every heading present, every claim with a source.
- A tool can check it too: a linter can refuse a research report with an empty Sources section.

| Without a template | With one |
|---|---|
| The answer may be in the last paragraph | The answer is the first heading |
| Sources are inline, partial, or missing | One line per source, with the date checked |
| Open questions are mixed into findings | Open questions have their own heading |

### The order that works

Every template on this page follows the same order, which is the order a reader needs:

1. The answer or decision, first.
2. How it was reached.
3. The detail that supports it.
4. What is still open.
5. Sources, with the date each was checked.

### Research report

```markdown
---
title: <question as a short label>
status: draft | complete
---
## Answer
<the answer in two to four sentences, first>

## Method
<what was searched or measured, and how>

## Findings
<one subsection per finding, each with its source>

## Open questions
<what is still unknown, and what would settle it>

## Sources
<one line per source with the date it was checked>
```

#### Worked example

A research agent asked "does the library support streaming responses?" fills it in. The Answer section reads "Yes, behind a flag: streaming is experimental and disabled by default." The Findings section quotes the documentation line. A lead agent reading only the Answer has the caveat, because the template put it first.

### Design document

```markdown
## Summary
<what is being built and why, in one paragraph>

## Decisions
<one subsection per decision: options, choice, reason>

## Plan
<numbered steps, each with how to tell it is done>

## Open questions
<each with who settles it>
```

#### Worked example

| Heading | Filled with |
|---|---|
| Summary | "Make every API date carry an explicit timezone, because two clients misread local times." |
| Decisions | "Store UTC, render with offset. Rejected: store local with zone name (harder to compare)." |
| Plan | "1. Add timezone to `parse_date` callers that lack it; done when the caller table has no 'no' rows." |
| Open questions | "Do report exports need the offset? Settled by the reports owner." |

### Bug report

```markdown
## Symptom
<what is wrong, as observed>

## Reproduction
<exact steps or command, and the output>

## Cause
<the line or condition, once known>

## Fix and regression test
<the change, and the test that fails without it>
```

The order matches the [bug-fix flow]({{ '/sections/12-testing/' | relative_url }}): reproduce, find the cause, freeze a failing test, fix.

### Comparing the kinds

| Kind | First heading | Read by | Checked for |
|---|---|---|---|
| Research report | Answer | Lead agent, person | Every finding has a source |
| Design document | Summary | Person, implementing agents | Every decision has a reason |
| Bug report | Symptom | Implementing agent, reviewer | A reproduction and a regression test |

## Goal templates

### What a goal condition is

`/goal` keeps a session working until a condition holds. After each turn, an evaluator reads the conversation and decides whether the condition is met. Two things follow from that:

- A vague condition never ends, or ends too early.
- The evaluator reads only the conversation, so the condition must name evidence the agent will print there.

### Vague and checkable

| Vague | Checkable |
|---|---|
| "Fix the tests" | "All tests in `tests/dates/` pass, shown by `pytest tests/dates/` output in this session, and the change is committed" |
| "Improve the docs" | "Every page under `docs/api/` has an Examples section, listed by a grep in this session" |
| "Finish the migration" | "No file under `src/` imports `old_client`, shown by a grep in this session, and the full test run passes" |

### The goal template

```markdown
Goal: <end state in one sentence>
Evidence the evaluator will see in this session:
- <command> prints <expected output>
- <command> prints <expected output>
Also done when: <a blocker is reported with its evidence and what would unblock it>
```

The last line gives the goal a second exit. Without it, a session that hits a blocker cannot meet the condition and keeps running.

#### Worked example

```markdown
Goal: every caller of parse_date passes an explicit timezone.
Evidence the evaluator will see in this session:
- `grep -rn "parse_date(" src/ | grep -v "tz="` prints nothing
- `pytest tests/dates/` ends with "passed" and no failures
Also done when: a caller cannot take a timezone without an API change;
report the caller and the change it needs.
```

<details markdown="1">
<summary>Why the evidence must be printed, not just true</summary>

The evaluator does not run commands and does not read the repository. If the agent fixed every caller but never ran the grep in the session, the evaluator has nothing to go on and the goal stays open. The template asks for the command and its output so the proof lands where the evaluator looks.

</details>

## Keeping document templates honest

Templates drift too: a heading nobody fills becomes noise, and a heading everyone needs gets added differently each time.

| Signal | What to change |
|---|---|
| A reader could not find something | Move it up, or give it a heading |
| An agent left the same heading empty three times | Drop the heading, or say in the template what goes there |
| Two documents of the same kind use different headings | Fix the template; a linter can then check the headings |
| A heading's content is always one line | Fold it into the heading above |

## Related pages

- [Sub-agent briefs]({{ '/sections/07-templates-and-stubs/briefs.html' | relative_url }}): the prompt stub, read once at launch.
- [Injected prompts]({{ '/sections/07-templates-and-stubs/injected-prompts.html' | relative_url }}): how an idle session with a goal is woken.
- [Logging and evaluation]({{ '/sections/16-logging-and-evaluation/' | relative_url }}): checking results over many runs.

## Sources

- `/goal` and its evaluator: <https://code.claude.com/docs/en/goal.md>, verified 2026-11-09.
- The document templates and goal template are from the author's own harness, described conceptually. The examples are illustrations written for this page.

{% include page_nav.html %}
