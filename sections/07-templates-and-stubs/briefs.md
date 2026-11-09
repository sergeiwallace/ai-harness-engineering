---
title: "Sub-agent briefs"
parent: "Templates, prompt stubs and injected prompts"
nav_order: 1
has_children: true
has_toc: false
redirect_from:
  - /sections/07-templates-and-stubs/detail.html
  - /sections/07-templates-and-stubs/advanced.html
last_updated: "2026-11-09"
talk_stop: "4"
stop_minutes: "6"
---

{% include page_nav.html %}

# Sub-agent briefs

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

### How a brief is sent

| Who starts the sub-agent | How | Where the brief goes |
|---|---|---|
| The lead agent, in Claude Code | Its Agent tool, naming a profile | The tool call's prompt is the brief |
| The lead, without naming a profile | The runtime picks a profile from the descriptions | Same |
| A person | An `@` mention of the sub-agent in a prompt | Claude writes the brief from the person's message |
| A skill that wraps the stub | `/<skill-name> args`, which fills the stub | The filled skill text is the brief |

The last row is the one that makes briefs consistent: a skill copies the fixed blocks every time. The worked example is on the overview: [a brief stub as a skill]({{ '/sections/07-templates-and-stubs/' | relative_url }}#worked-example-a-brief-stub-as-a-skill).

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

<!-- IMAGE NEEDED: an annotated prompt or brief with each section labelled in the margin; search: annotated prompt template, prompt anatomy diagram, labelled form fields -->

Each field below has what it holds and what goes wrong without it:

- Filled for the task: Objective, Inputs, Output, Tools and scope, When stuck.
- Fixed text from the stub: Boundaries.

### Objective

#### What it holds

One sentence naming the end state.

| An activity (weak) | An end state (strong) |
|---|---|
| "Look into the date parsing" | "A table of every caller of `parse_date`, marked by whether it passes a timezone" |
| "Look into the flaky test" | "Name the cause of the `test_retry` failure, with the line that causes it" |
| "Improve error handling" | "Every call to `fetch` in `client/` handles a timeout, with a test per call site" |
| "Review this PR" | "A list of bugs in the diff, each with file, line, severity and a one-line fix" |

#### What goes wrong without it

- The agent stops at the first plausible result.
- Or it keeps going until it runs out of turns.
- In both cases it has no way to tell which is right; the objective gives it one.

### Inputs

#### What it holds

- The files, links, branches and data the sub-agent starts from.
- What the parent already learned about them.
- What the parent already ruled out.

#### What goes wrong without it

- The sub-agent spends its first minutes finding what the parent already found.
- It may start from a different place: the old copy of a file, the wrong branch.
- It may repeat the parent's dead ends.

> Tip: name what the parent already ruled out. "The config loader is not the cause; it was checked" saves a sub-agent from repeating the parent's dead end.

### Output

#### What it holds

- The exact format of the result.
- Where to put it: the final answer, or a file at a named path.
- A length cap: rows, words or lines.

#### What goes wrong without it

1. The parent starts five sub-agents and gets back five formats.
2. It spends its own context reconciling them.
3. A sub-agent with no cap pastes the files it read into its answer, which fills the parent's window and undoes the reason for delegating.

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

#### What goes wrong without it

| Missing | What happens |
|---|---|
| A read-only statement | A researcher edits a file by accident |
| A named repository | A writer edits the wrong checkout |
| A directory | The agent wanders the whole repository and fills its context |

The brief states the scope; the [agent profile]({{ '/sections/08-delegation/agent-profiles.html' | relative_url }}) enforces the tool part of it, because a profile with no edit tools cannot edit whatever the brief says.

### Boundaries

#### What it holds

The safety rules, word for word, the same in every brief.

```markdown
## Boundaries (fixed, do not edit)
- Never kill processes by name pattern; kill one PID or one process group.
- No recursive force deletes. Confirm every deletion against its target.
- Keep scratch files in your own temporary directory.
- Never print secrets, including through a redirect or shell tracing.
- Never skip the commit hooks.
```

#### Why it is fixed text

- A paraphrase loses detail.
- A pointer ("follow the safety rules in the docs") delivers nothing, because the sub-agent may never open the docs.
- Fixed text can be checked: a launch guard compares the block with the stub byte for byte.

#### Where it sits among the layers

The Boundaries block is the instruction layer. A model applies instructions inconsistently, so each rule in it is also enforced in code:

| Rule in the block | Where it is enforced | What the brief line adds |
|---|---|---|
| No pattern kills | A hook refuses the command | The agent knows why it was refused |
| No recursive force deletes | A hook refuses the command | Same |
| Scratch in your own temp directory | A guard checks paths | Same |
| Never skip commit hooks | The commit tool refuses the flag | Same |

When a rule is enforced in code, its brief line can shrink to a short reminder. Adding a new detailed line to the block after every mistake makes the brief longer without making the rule more reliable. See [keeping templates small]({{ '/sections/07-templates-and-stubs/' | relative_url }}#keeping-templates-small).

### When stuck

#### What it holds

- What to try when the obvious path fails.
- When to stop and report instead.
- How to mark a partial result.

```markdown
## When stuck
If a test fails for a reason unrelated to the change, record it and continue.
If the same command fails three times, stop and report the command and its output.
Stop after 30 minutes and report what you have, marked partial.
```

#### What goes wrong without it

| Without the field | With it |
|---|---|
| The agent loops on the same failing attempt | It stops after three tries and reports the command |
| It guesses and reports the guess as a result | It reports what it has, marked partial |

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

Checks 1 to 3 are facts a script can test. A guard in the launch path runs them every time:

```python
# brief_check.py: refuse a brief whose fixed blocks differ from the stub
import hashlib, re, sys

def block(text, heading):
    m = re.search(rf"^## {re.escape(heading)}.*?\n(.*?)(?=^## |\Z)", text, re.M | re.S)
    return m.group(1).strip() if m else None

brief = open(sys.argv[1], encoding="utf-8").read()
stub = open(sys.argv[2], encoding="utf-8").read()
for heading in ("Objective", "Inputs", "Output", "Tools and scope", "Boundaries", "When stuck"):
    if block(brief, heading) is None:
        sys.exit(f"refused: missing field {heading}")
if block(brief, "Boundaries") != block(stub, "Boundaries"):
    sys.exit("refused: Boundaries block differs from the stub")
print("brief ok")
```

Checks 4 and 5 are judgment, so a reader makes them, or the lead before it sends the brief.

<details markdown="1">
<summary>Why a guard checks the fixed blocks and leaves the filled ones to a reader</summary>

- The filled fields are judgment: whether an objective is clear enough is a question for a reader.
- The fixed blocks are facts: either the boundaries are the stub's exact text or they are not.
- A guard that checks only facts never refuses a good brief for a style reason, so people and agents do not learn to route around it.

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

- Sub-agent context, profiles and `@` mentions: [Claude Code docs: sub-agents](https://code.claude.com/docs/en/sub-agents.md), verified 2026-11-09.
- Skills and argument placeholders: [Claude Code docs: skills](https://code.claude.com/docs/en/skills.md), verified 2026-11-09.
- The brief stub, its fixed boundaries and the brief check are from the author's own harness, described conceptually. The examples are illustrations written for this page.

{% include page_nav.html %}
