---
title: "Sub-agent briefs"
parent: "Templates, prompt stubs and injected prompts"
nav_order: 1
has_children: true
has_toc: false
redirect_from:
  - /sections/07-templates-and-stubs/detail.html
  - /sections/07-templates-and-stubs/advanced.html
---

{% include page_nav.html %}

# Sub-agent briefs

As of 2026-11-09. Topic page in [Templates, prompt stubs and injected prompts]({{ '/sections/07-templates-and-stubs/' | relative_url }}). Slides: [talk stop 4]({{ '/talk/' | relative_url }}#stop-4-6-minutes).

A brief is the prompt one agent writes to start another. It is the only channel from parent to child: the sub-agent starts with an empty context and knows what the brief tells it, plus its profile and the project's context files. This page covers each field of a brief in depth, a filled brief from end to end, how to review one, and the way briefs fail. The person's original words, which sit at the top of every brief, have their own page: [the original-request block]({{ '/sections/07-templates-and-stubs/original-request-block.html' | relative_url }}).

![A sequence across five participants. The person sends a request in exact words to the lead agent, which stores them with a hash. The lead sends a brief filled from the stub to the launch guard, which refuses it because the request block was edited. The lead resends with the block copied unchanged and the guard launches the sub-agent with the brief only. The sub-agent reads and searches the repository, reads thirty files and returns a nine-row table under the cap. The lead re-runs the search itself, gets the same nine callers, and returns the result to the person, checked against the request.]({{ '/assets/images/07-templates-and-stubs/brief-in-flight.png' | relative_url }}){: style="width:100%"}

*Figure 1. A brief in flight, from request to checked result. Built with archify.*

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## What a brief is for

### The sub-agent's starting context

| In the parent session | Reaches the sub-agent? |
|---|---|
| The brief it writes | Yes, in full |
| The project's context files | Yes, loaded fresh |
| The sub-agent's own profile | Yes, as standing instructions |
| The conversation history | No |
| Corrections the person made this session | No, unless written into the brief |
| Rules the parent learned an hour ago | No, unless written into the brief |

In figure 1 the launch guard checks the brief's fixed blocks because they are the only route by which the parent's rules reach the sub-agent.

### Stub, then brief

A stub is the template; a brief is one filled copy.

1. The lead copies the stub.
2. It fills the angle-bracket fields for this task.
   - Objective, inputs, output, tools and scope, when stuck.
3. It leaves the fixed text alone.
   - The original-request block and the Boundaries block.
4. The launch guard checks the fixed text before the sub-agent starts.

Two briefs from the same stub differ only in what they are about, so a reviewer can compare any brief with the stub line by line.

## The six fields

### Objective

#### What it holds

One sentence saying what done looks like. Not the activity ("look into the date parsing"), the end state ("a table of every caller of `parse_date`, marked by whether it passes a timezone").

#### What goes wrong without it

An agent with no end state either stops at the first plausible result or keeps going until it runs out of turns. The objective lets it tell which.

| Weak objective | Strong objective |
|---|---|
| Look into the flaky test | Name the cause of the `test_retry` failure, with the line that causes it |
| Improve error handling | Every call to `fetch` in `client/` handles a timeout, with a test per call site |
| Review this PR | A list of bugs in the diff, each with file, line, severity and a one-line fix |

### Inputs

#### What it holds

The files, links, branches and data the sub-agent starts from, and anything the parent already learned about them.

#### What goes wrong without it

Without inputs the sub-agent spends its first ten minutes finding what the parent already found. Worse, it may find a different starting point: the old copy of a file, the wrong branch.

> Tip: name what the parent already ruled out. "The config loader is not the cause; it was checked" saves a sub-agent from repeating the parent's dead end.

### Output

#### What it holds

The exact format of the result, where to put it, and a length cap.

#### What goes wrong without it

The parent reads the output of every sub-agent it starts. If five sub-agents return five formats, the parent spends its own context reconciling them. A fixed format lets it read each result the same way, and a length cap keeps a sub-agent from pasting the hundred files it read back into the parent's window, which defeats the reason for delegating.

#### Formats that work

| Need | Format |
|---|---|
| A list of findings | Markdown table with fixed columns |
| A decision | The answer first, in two sentences, then the evidence |
| Code changes | Commit ids and the exact test command with its last line of output |
| Research | A file at a named path, plus a three-line summary as the final answer |

### Tools and scope

#### What it holds

- Which tools the sub-agent may use: read and search only, or edit and shell too.
- Which directories or repositories it may touch.
- Whether it may write at all, and where: its own worktree only.

```markdown
## Tools and scope
Read and search only. Directory: src/dates/ and its tests.
No edits, no shell commands that change files.
```

#### What it holds

Which tools the sub-agent may use, which directories or repositories it may touch, and whether it may write at all.

#### What goes wrong without it

Scope limits what can go wrong. A researcher with write access can edit a file by accident; a writer with no stated repository can write into the wrong checkout. The brief states the scope, and the [agent profile]({{ '/sections/08-delegation/agent-profiles.html' | relative_url }}) enforces it: a profile with no edit tools cannot edit whatever the brief says.

### Boundaries

#### What it holds

The safety rules, word for word, the same in every brief.

#### Why it is fixed text

This is the field that carries the rules the sub-agent has no other source for. A paraphrase loses detail; a pointer ("follow the safety rules in the docs") delivers nothing. The stub holds the exact sentences, and the lead agent never edits them.

```markdown
## Boundaries (fixed, do not edit)
- Never kill processes by name pattern; kill one PID or one process group.
- No recursive force deletes. Confirm every deletion against its target.
- Keep scratch files in your own temporary directory.
- Never print secrets, including through a redirect or shell tracing.
- Never skip the commit hooks.
```

#### Keeping the block honest

Every line in the Boundaries block exists because something went wrong once. The block is the instruction layer; the enforcing happens elsewhere.

| Rule in the block | Where it is enforced | What the brief line adds |
|---|---|---|
| No pattern kills | A hook refuses the command | The agent knows why it was refused |
| No recursive force deletes | A hook refuses the command | Same |
| Scratch in your own temp directory | A guard checks paths | Same |
| Never skip commit hooks | The commit tool refuses the flag | Same |

When a hook or guard enforces a rule, the brief line can shrink to a short reminder or leave the brief. See [keeping templates small]({{ '/sections/07-templates-and-stubs/' | relative_url }}#keeping-templates-small).

### When stuck

#### What it holds

What to try when the obvious path fails, and when to stop and report instead.

#### What goes wrong without it

A sub-agent that hits a wall has two bad options without this field: loop on the same attempt, or guess and report the guess as a result. "When stuck" gives it a third: a stated fallback and a stated stop.

```markdown
## When stuck
If a test fails for a reason unrelated to the change, record it and continue.
If the same command fails three times, stop and report the command and its output.
Stop after 30 minutes and report what you have, marked partial.
```

## A filled brief, end to end

### The task

A lead agent is planning a change to date handling and needs to know who calls the parser and how.

- Repository: one service, about 200 Python files.
- What the lead already knows: the CLI wrapper calls `parse_date` once, with `tz=UTC`.
- What it needs back: a list it can turn into engineer briefs.

### The brief

```markdown
## Original request (verbatim, do not edit)
> make every date in the API responses carry an explicit timezone
record: asks/dates-tz.md  sha256: 41ab...07c2

## Objective
Find every caller of `parse_date` and list which pass a timezone.

## Inputs
The repository at the current branch. Start from src/dates.py.
Already checked: the CLI wrapper calls it once with tz=UTC.

## Output
A Markdown table: file, line, passes timezone (yes or no). At most 40 rows.
Write it as your final answer; do not create files.

## Tools and scope
Read and search only. No edits, no shell commands that change files.

## Boundaries (fixed, do not edit)
- Never kill processes by name pattern; kill one PID.
- Never cite a file that is not tracked in git.

## When stuck
If a caller is generated code, list it once with "generated" and move on.
Stop after 30 minutes and report what you have.
```

### What came back

```text
| file                   | line | passes timezone |
|------------------------|------|-----------------|
| src/api/orders.py      | 88   | no              |
| src/api/orders.py      | 141  | yes             |
| src/reports/daily.py   | 23   | no              |
| src/gen/schema_pb2.py  | -    | generated       |
... 9 rows total
```

The parent's context gains the nine-row table. The thirty files stay in the sub-agent's context, which ends when the sub-agent does.

## Reviewing a brief

A brief can be checked before launch, by the lead agent itself or by a guard. The check is mechanical, which is the advantage of a template.

1. Every field heading is present.
2. The Boundaries block matches the stub byte for byte.
3. The original-request block matches its stored hash.
4. The output names a format and a cap.
5. The scope names a directory or repository, and says read-only or not.

<details markdown="1">
<summary>Why a guard checks the fixed blocks, not the filled ones</summary>

The filled fields are judgment: whether an objective is clear enough is a question for a reader. The fixed blocks are facts: either the boundaries are the stub's exact text or they are not. A guard that checks only facts never refuses a good brief for a style reason, so people and agents do not learn to route around it.

</details>

## How briefs fail

| Failure | What it looks like | Fix |
|---|---|---|
| Pointer instead of rule | "Follow the testing rules" | Enforce the rule with a hook or test, and name it in Boundaries as the second layer |
| No stop | The sub-agent runs until its turn limit | A time or attempt cap in When stuck |
| Uncapped output | The parent's window fills with pasted files | A row or word cap in Output |
| Paraphrased request | A requirement silently dropped | [The original-request block]({{ '/sections/07-templates-and-stubs/original-request-block.html' | relative_url }}) |
| Brief bloat | Every incident adds a paragraph | Prune; move enforced rules into hooks |

## Related pages

- [The original-request block]({{ '/sections/07-templates-and-stubs/original-request-block.html' | relative_url }}), the fixed block at the top of every brief.
- [Agent profiles]({{ '/sections/08-delegation/agent-profiles.html' | relative_url }}), which enforce what the brief's scope states.
- [Checking delegated work]({{ '/sections/08-delegation/checking-delegated-work.html' | relative_url }}), what to do with what comes back.

## Sources

- Sub-agent context and profiles: <https://code.claude.com/docs/en/sub-agents.md>, verified 2026-11-09.
- The brief stub, its fixed boundaries and the brief check are from the author's own harness, described conceptually. The examples are illustrations written for this page.

{% include page_nav.html %}
