---
title: "Skill anatomy and loading"
parent: "Skills, commands and plugins"
nav_order: 1
has_children: true
has_toc: false
redirect_from:
  - /sections/05-skills-and-plugins/detail.html
  - /sections/05-skills-and-plugins/advanced.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Skill anatomy and loading

This page takes a skill apart: the folder, the `SKILL.md` file, each frontmatter field, how the client loads the parts, how arguments and live command output reach the body, and the open standard that lets one skill run in more than one client. It is part of [Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }}). The sub-page [Writing skill descriptions]({{ '/sections/05-skills-and-plugins/skill-descriptions.html' | relative_url }}) covers the one field that decides whether a skill is ever used.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The folder

A skill is a folder. The only required file is `SKILL.md`. Everything else in the folder is there because a step in `SKILL.md` names it.

```text
.claude/skills/<name>/       # here <name> is release
├── SKILL.md              # frontmatter + steps (required)
├── checklist.md          # reference: read at step 2
├── templates/
│   └── changelog.md      # template: filled in at step 4
└── tools/
    └── bump_version.py   # script: run at step 3
```

### What each kind of file is for

| Kind | Loaded into the window | Typical content | Rule of thumb |
|---|---|---|---|
| `SKILL.md` | When the skill is used | The steps, in order | Keep it under a few hundred lines |
| Reference file | When a step reads it | Checklists, policies, long examples | Move anything only one step needs here |
| Template | When a step fills it | A document skeleton | The skill says where to save the result |
| Script | Never; only its output | Deterministic work: parse, count, bump | Anything a program can do exactly goes here |

#### Why scripts matter

A script is the part of a skill that a program does exactly, which makes it the enforcing part of the procedure (layer 2 of the [four layers]({{ '/sections/02-what-is-a-harness/' | relative_url }})).

| Step written as | What happens |
|---|---|
| "Count the open issues and sort them by age" | The model does the counting in prose, and the answer can differ between runs |
| "Run `tools/open_issues.py --sort age`" | The same answer every run, and only the output enters the window |

Put deterministic work in scripts and keep judgment in the steps.

### Where skills live

| Level | Path | Shared with | Use for |
|---|---|---|---|
| User | `~/.claude/skills/<name>/` | Only you, every project | Your own working habits |
| Project | `.claude/skills/<name>/` | Everyone who clones the repo | The team's procedures |
| Plugin | `<plugin>/skills/<name>/` | Everyone who installs the plugin | Procedures shared across repos |

Give skills at different levels different names. When two levels define the same name, the client picks one by a fixed precedence order in its docs, and the person who expected the other one gets a different procedure.

## The SKILL.md file

`SKILL.md` is YAML frontmatter between two `---` lines, then Markdown instructions.

### The frontmatter fields

| Field | Required | What it does |
|---|---|---|
| `name` | Yes | The skill's name and its slash command. Lowercase letters, digits and hyphens, up to 64 characters in the Agent Skills specification |
| `description` | Yes | What the skill does and when to use it. Always in the window. Up to 1,024 characters in the specification |
| `disable-model-invocation` | No | `true`: only a person can start it |
| `user-invocable` | No | `false`: only the model can start it; hidden from the menu |
| `allowed-tools` | No | Tools the skill may use without asking for permission while it runs |
| `argument-hint` | No | Shown in the menu, such as `[issue-number]` |
| `model` | No | Run the skill on a named model |
| `context` | No | `fork` runs the skill in a separate sub-agent context |

The two length limits come from the Agent Skills specification and were not re-checked for this page; a client can set its own. The [invocation page]({{ '/sections/05-skills-and-plugins/invocation-and-chaining.html' | relative_url }}#who-can-start-a-skill) covers the two invocation fields in depth. The [descriptions sub-page]({{ '/sections/05-skills-and-plugins/skill-descriptions.html' | relative_url }}) covers `description`.

#### Example: a full frontmatter block

```yaml
---
name: fix-issue
description: Fix a GitHub issue end to end on a new branch. Use when the user gives an issue number and asks for a fix.
argument-hint: "[issue-number]"
allowed-tools: Bash(gh issue view:*), Bash(git checkout:*), Bash(npm test:*)
---
```

- `allowed-tools` lets the skill read the issue, make a branch and run the tests without a permission prompt at each call.
- Anything outside that list still asks.
- Keep the list narrow: `Bash(npm test:*)` allows the test command, where `Bash` alone would allow every shell command.

### The body

The body is instructions for the agent, written as steps. Three habits make a body reliable:

1. Number the steps, so the agent and the person can both say where it is.
2. Name the exact command or file at each step. "Run the tests" is weaker than "run `npm test -- --coverage`".
3. End with what to report and where to save it, so the result is in a known place.

> A body is read on every use, so its length is a cost each time. If a section is needed only at one step, move it to a reference file and have that step read it.

## How loading works

### The three layers

![Three stacked layers. Layer 1, the name and description of every skill, is always loaded. Layer 2, the SKILL.md body, loads on use. Layer 3, files in the skill folder, load on demand.]({{ '/assets/images/05-skills-and-plugins/skill-loading-layers.svg' | relative_url }})

Anthropic calls this progressive disclosure: the client shows the model only as much of a skill as the current step needs.

1. Layer 1, the name and description, is in the window on every turn for every model-invocable skill.
2. Layer 2, the body, enters when the skill starts.
3. Layer 3, a file in the folder, enters when a step reads it; a script contributes only its output.

### The sequence

![Sequence diagram with five lanes: Person, Client, Context window, Model and Skill folder. At session start the client puts every skill's name and description into the window. The person types /release 2.4.0; the client reads SKILL.md and the body enters the window. At step 2 the model reads checklist.md; at step 3 it runs bump_version.py and only the output enters the window. The model reports back.]({{ '/assets/images/05-skills-and-plugins/skill-loading-sequence.png' | relative_url }})
*The release skill from the folder above, during a real run. Each arrow into the context window is a moment the window grows.*

#### What this means for size

| Part | Paid | Budget it as |
|---|---|---|
| Description | On every turn, for every skill | One or two sentences |
| Body | Once per use | The steps, nothing else |
| Reference file | Once per step that reads it | As long as it needs to be |
| Script | Its output only | Make the output short |

### Running in a forked context

`context: fork` runs the skill in its own sub-agent context.

- The skill's reading and tool output stay in that context.
- Only its final result returns to the main conversation.
- Use it for a skill that reads a lot to produce a little, such as a review that reads 40 files and returns ten findings.

![Sequence diagram with three lanes: Main conversation, Forked context and Repository. The main conversation starts /review on the current diff in a forked context. The fork runs git diff and gets 12 changed files, then reads callers and tests, 28 more files, so 40 files sit in the fork's window. The fork returns 10 findings, one paragraph each, to the main conversation, which then fixes the two high findings.]({{ '/assets/images/05-skills-and-plugins/forked-review-sequence.png' | relative_url }})
*A forked review. The 40 files stay in the fork's window; the main conversation receives only the findings.*

<details markdown="1">
<summary>When a fork is the wrong choice</summary>

A forked skill cannot see the conversation that came before it except what is passed in its arguments. A skill that depends on decisions made earlier in the session, such as "write the PR description for what we just did", should run in the main context. A skill that starts from the repository state alone, such as "review the current diff", forks well.

</details>

## Arguments and live data

### Arguments

Whatever follows the command is passed to the body, where `$ARGUMENTS` stands for all of it.

```markdown
---
name: fix-issue
description: Fix a GitHub issue end to end. Use when the user gives an issue number.
argument-hint: "[issue-number]"
---
1. Read the issue: `gh issue view $ARGUMENTS`.
2. Create a branch named `fix/$ARGUMENTS`.
3. Write a failing test that shows the bug, then fix it.
4. Run the tests and report the last line of output.
```

Typing `/fix-issue 412` makes step 1 read `gh issue view 412`.

#### Edge cases

| Typed | `$ARGUMENTS` | What to write in the skill |
|---|---|---|
| `/fix-issue 412` | `412` | Nothing extra |
| `/fix-issue` | Empty | A step that says what to do with no number, such as "ask for the issue number" |
| `/fix-issue 412 and add a test` | `412 and add a test` | Treat the first word as the number and the rest as notes, and say so in the step |

### Injected command output

A line of the form `` !`command` `` runs the command when the skill loads, and its output replaces the line. The agent starts with the data already in front of it.

#### Example: a standup skill

```markdown
---
name: standup
description: Draft my standup update from yesterday's work. Use when the user asks for a standup.
disable-model-invocation: true
---
Commits since yesterday:
!`git log --since=yesterday --author="$(git config user.email)" --oneline`

Open pull requests:
!`gh pr list --author @me --json title,url --jq '.[] | "\(.title) \(.url)"'`

Write three short lines: done, next, blocked. Use only the data above.
```

#### What the model receives

```text
Commits since yesterday:
a41c2e9 fix: retry the upload when the token expires
77b0d13 test: cover the expired-token path

Open pull requests:
Retry expired uploads https://github.com/example/app/pull/88

Write three short lines: done, next, blocked. Use only the data above.
```

> Injected commands run with your permissions when the skill loads. Treat a skill from someone else like a script from someone else: read it before you install it.

## The Agent Skills standard

Skills follow the Agent Skills open standard.

- The format: a folder with a `SKILL.md`, a name and a description in frontmatter, and Markdown instructions.
- Governance: Anthropic gave the standard to the Agentic AI Foundation in October 2026.
- Clients: Codex reads the same format, from `.agents/skills`.

### Moving a skill between clients

| | Claude Code | Codex |
|---|---|---|
| Project folder | `.claude/skills/` | `.agents/skills/` |
| Required fields | `name`, `description` | `name`, `description` |
| Turn off model invocation | `disable-model-invocation: true` | `allow_implicit_invocation: false` |
| Client-specific extras | `allowed-tools`, `context`, `` !`command` `` | Its own settings |

The core moves unchanged. Client-specific fields are ignored by the other client or need a small rewrite, so a skill meant for both keeps them few.

#### Example: one skill, two clients

1. The team keeps its `bug-report` skill in one folder.
2. It links that folder into both `.claude/skills/` and `.agents/skills/`.
3. The skill uses only `name`, `description` and numbered steps, so both clients run it the same way.
4. A fix to the one folder reaches both clients.

## Checking a skill works

A skill is code written in prose, so check it the way you would check a script:

1. Start a new session, so the window holds the current description.
2. Ask for the task in the words a person would use, without typing the command, and confirm the model picks the skill.
3. Type the command with and without arguments, and read the result of each step.
4. Run the scripts in the folder on their own, with tests.
5. Add the skill to a frontmatter test such as the [one on the overview]({{ '/sections/05-skills-and-plugins/' | relative_url }}#a-frontmatter-test).

## Pitfalls

- A body that tries to hold every case grows past the point where the agent follows it. Split it, or move cases into reference files.
- Work a script could do exactly, left to the model, gives a different answer each run.
- A forked skill that needs earlier conversation produces a confident wrong result.
- A skill from a stranger runs its injected commands as you.
- An edited `SKILL.md` is not used until `/reload-skills` or a new session.

## Sources

- Skills, frontmatter fields, arguments, injected command output and forked context: <https://code.claude.com/docs/en/skills.md>, verified 2026-11-09.
- Agent Skills donation: <https://www.anthropic.com/news/donating-agent-skills-to-the-agentic-ai-foundation>, verified 2026-11-08.
- Codex skills: <https://learn.chatgpt.com/docs/build-skills.md>, verified 2026-11-08.
- The example skills and transcript are illustrations written for this page. The loading-layers diagram was drawn for this page.

{% include page_nav.html %}
