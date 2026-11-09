---
title: "Document and goal templates"
parent: "Templates, prompt stubs and injected prompts"
nav_order: 3
---

{% include page_nav.html %}

# Document and goal templates

As of 2026-11-09. Topic page in [Templates, prompt stubs and injected prompts]({{ '/sections/07-templates-and-stubs/' | relative_url }}).

A document template is the full shape of one kind of document, with a description of what goes under each heading. A goal template is the shape of a stop condition for `/goal`. Prompt stubs are read once, at launch, by one agent; these are read later, by many readers, or after every turn by an evaluator. This page covers both:

- Document templates for a research report, a design document and a bug report, with a worked example each and a check that enforces the headings in code.
- Goal templates: how `/goal` evaluates a condition, its commands and limits, and how to write a condition the evaluator can check.

![A lifecycle from left to right: goal armed, agent works, turn ends, evaluator reads, goal met. From the evaluator, not yet loops back to agent works, and a stated blocker leads to a blocker-reported state. From agent works, a started job leads to an idle state that returns on an injected wake-up.]({{ '/assets/images/07-templates-and-stubs/goal-evaluator.png' | relative_url }}){: style="width:100%"}

*Figure 1. A goal from start to stop: the evaluator reads the conversation after every turn. The "injected wake-up" arrow is the case of a job outside the session; for the session's own background work, Claude Code delivers the result as a new turn itself (see [injected prompts]({{ '/sections/07-templates-and-stubs/injected-prompts.html' | relative_url }}#goal-check-ins)). Built with archify.*

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

A research agent is asked "does the library support streaming responses?" and fills in the template:

| Heading | Filled with |
|---|---|
| Answer | "Yes, behind a flag: streaming is experimental and disabled by default." |
| Method | "Read the library's streaming guide and its changelog for the current version." |
| Findings | The documentation line quoted, with its link |
| Open questions | "Is the flag planned to become default? The changelog does not say." |
| Sources | Two links, each with the date checked |

A lead agent reading only the Answer has the caveat, because the template put it first.

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

### Checking a document against its template

A template written down is an instruction; a check in code enforces it. A short script can refuse a document whose headings do not match its kind:

```python
# check_headings.py: fail when a research report is missing a required heading
import re, sys

REQUIRED = {"research": ["Answer", "Method", "Findings", "Open questions", "Sources"]}

def missing(path, kind):
    text = open(path, encoding="utf-8").read()
    found = set(re.findall(r"^## (.+)$", text, re.M))
    return [h for h in REQUIRED[kind] if h not in found]

gaps = missing(sys.argv[1], sys.argv[2])
if gaps:
    sys.exit(f"{sys.argv[1]}: missing headings: {', '.join(gaps)}")
```

1. An agent writes a research report from the template.
2. A commit hook, or a CI step, runs the check on every changed report.
3. A report missing a heading is refused with the heading names, so the agent fixes it before the change lands.

The same check can test the order of headings, or that the Sources section has at least one line.

## Goal templates

### What a goal is

`/goal` sets a completion condition, and Claude keeps working toward it without a prompt for each step.

1. You run `/goal` followed by the condition. Setting it starts a turn at once, with the condition as the directive.
2. After each turn, Claude Code sends the condition and the conversation so far to the configured small fast model.
3. That evaluator returns one of three verdicts, each with a short reason:
   - Not yet met: Claude keeps working and takes the reason as guidance.
   - Met: the goal clears and an achieved entry is recorded.
   - Impossible: the goal clears and a failed entry is recorded with the reason.
4. If Claude keeps answering the evaluator without making progress (no tool use for several turns in a row), Claude Code stops the loop and returns control with the goal still set.

### The commands

```text
/goal all tests in tests/dates pass and the lint step is clean   # set (replaces any active goal)
/goal                                                             # status: condition, time, turns, tokens, last reason
/goal clear                                                       # remove; stop, off, reset, none and cancel also work
```

```bash
# non-interactive: runs the loop to completion in one invocation
claude -p "/goal CHANGELOG.md has an entry for every PR merged this week" --output-format stream-json --verbose
```

| Fact | Value |
|---|---|
| Goals per session | One |
| Condition length | Up to 4,000 characters |
| Survives a resume | Yes; the turn count, timer and token baseline reset |
| Who evaluates | The small fast model for your provider; `ANTHROPIC_DEFAULT_HAIKU_MODEL` changes it |
| What the evaluator can see | Only the conversation; it calls no tools |

### Writing a checkable condition

The evaluator does not run commands or read files, so the condition must name evidence that Claude's own output will show. Claude Code's documentation lists three parts of a condition that holds up:

| Part | Example |
|---|---|
| One measurable end state | "every caller of `parse_date` passes a timezone" |
| A stated check | "`pytest tests/dates/` exits 0" |
| Constraints that matter | "no other test file is modified" |

A turn or time clause, such as "or stop after 20 turns", bounds how long a goal runs; Claude reports progress against it each turn.

#### Vague and checkable

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
Constraints: <what must not change on the way>
Bound: or stop after <N> turns and report progress
Also done when: <a blocker is reported with its evidence and what would unblock it>
```

| Line | Why it is there |
|---|---|
| Evidence | The evaluator reads only the conversation |
| Constraints | Without them, the goal can be met by breaking something else |
| Bound | Caps the run if progress stalls |
| Also done when | Names the blocker case, so the evaluator can return "impossible" with a reason the person can act on |

#### Worked example

```markdown
Goal: every caller of parse_date passes an explicit timezone.
Evidence the evaluator will see in this session:
- `grep -rn "parse_date(" src/ | grep -v "tz="` prints nothing
- `pytest tests/dates/` ends with "passed" and no failures
Constraints: no change to the public signature of parse_date.
Bound: or stop after 25 turns and report progress.
Also done when: a caller cannot take a timezone without an API change;
report the caller and the change it needs.
```

<details markdown="1">
<summary>Why the evidence must be printed, not just true</summary>

- The evaluator does not run commands and does not read the repository.
- If the agent fixed every caller but never ran the grep in the session, the evaluator has nothing to go on and the goal stays open.
- The template asks for the command and its output so the proof lands where the evaluator looks.

</details>

### When a goal is the wrong tool

| Need | Better fit |
|---|---|
| Re-run a prompt on a time interval | `/loop` |
| A deterministic check after every turn, in every session | A Stop hook in settings that runs a script |
| Work independent of any open session, such as nightly tests | A scheduled task |

#### Edge cases

| Case | What happens |
|---|---|
| A sub-agent or background shell is still running when the turn ends | Evaluation is skipped until a turn ends with no background work running |
| A turn fails on an error you must fix (authentication, exhausted credit, unrecoverable context overflow, unavailable model) | The goal clears; set it again after the fix |
| A rate limit or usage limit | The goal pauses and resumes later |
| Hooks are disabled (`disableAllHooks`) or limited to managed hooks | `/goal` is unavailable, because the evaluator is a prompt-based Stop hook |

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
- [Injected prompts]({{ '/sections/07-templates-and-stubs/injected-prompts.html' | relative_url }}): goal check-ins and the other routes that wake an idle session.
- [Logging and evaluation]({{ '/sections/16-logging-and-evaluation/' | relative_url }}): checking results over many runs.

## Sources

- `/goal` commands, evaluator, verdicts, condition advice, limits and edge cases: <https://code.claude.com/docs/en/goal.md>, verified 2026-11-09.
- The document templates and goal template are from the author's own harness, described conceptually. The examples are illustrations written for this page.

{% include page_nav.html %}
