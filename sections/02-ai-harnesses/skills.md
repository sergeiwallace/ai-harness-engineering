---
title: "Harness skills"
parent: "AI harnesses"
nav_order: 2
---

{% include page_nav.html %}

# Harness skills

As of 2026-11-09. Part of [AI harnesses]({{ '/sections/02-ai-harnesses/' | relative_url }}). The full section on this topic is [Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }}).

A skill is a procedure packaged so the agent can follow it on demand. It is how a prompt you keep retyping becomes a step-by-step method that runs the same way every time, without paying for its full text on every turn.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Anatomy of a skill

### The folder

```text
.claude/skills/<name>/
  SKILL.md          front matter and instructions
  reference.md      detail the body points to
  templates/        files the skill fills in
  check.py          a script a step runs
```

### Where skills live

| Scope | Claude Code | Codex |
|---|---|---|
| you, every project | `~/.claude/skills/<name>/SKILL.md` | `$HOME/.agents/skills/<name>/SKILL.md` |
| one project, shared | `.claude/skills/<name>/SKILL.md` | `.agents/skills/<name>/SKILL.md`, in any folder from the working folder up to the repository root |
| organisation | the managed settings folder | `/etc/codex/skills` |
| a plugin | `<plugin>/skills/<name>/SKILL.md` | bundled skills such as `skill-creator` |
| start it by hand | `/name` | `$name` in the prompt, or `/skills` |

Both tools build on the open Agent Skills format, so a skill with a `name` and a `description` works in both. Each adds its own optional fields.

### The front matter

```markdown
---
name: release-notes
description: Draft release notes from merged changes since the last tag. Use when asked for release notes or a changelog.
argument-hint: "[since-tag]"
allowed-tools: Read Grep Bash(git log *) Bash(git tag *)
---
```

Claude Code's main fields:

| Field | Does |
|---|---|
| `name` | the `/name` command; defaults to the folder name |
| `description` | what the skill does and when to use it; the model reads it on every turn to decide when to start the skill |
| `when_to_use` | extra trigger phrases; with `description`, cut at 1,536 characters in the listing |
| `argument-hint` | shown while typing, such as `[issue-number]` |
| `arguments` | named arguments, used as `$name` in the body |
| `allowed-tools` | tools the skill may use without a permission prompt during the turn it runs |
| `disallowed-tools` | tools removed while the skill is active |
| `disable-model-invocation` | `true`: only the person can start it |
| `user-invocable` | `false`: only the model can start it |
| `context: fork` | runs the skill in a forked sub-agent, with `agent` naming which one |
| `paths` | glob patterns that limit when the model loads it |
| `model`, `effort` | model and effort for the rest of the turn |

> `allowed-tools` grants; it does not restrict. To keep a skill away from a tool, use `disallowed-tools` or a permission deny rule.
{: .warning }

### Progressive loading

![A skill loads in three stages: name and description always, the SKILL.md body on use, and reference files and scripts per step.]({{ '/assets/images/02-ai-harnesses/skill-loading.svg' | relative_url }})

1. Every turn: only the name and description sit in the context.
2. On use: the rendered `SKILL.md` body enters the conversation as one message, and stays there for later turns of the session.
3. Per step: reference files load when a step reads them, and scripts run without their source entering the context.

This is what lets a harness carry many skills without crowding the window. Once used, a long skill body does take room for the rest of the session, which is one more reason to keep it short.

## How a skill triggers

### By the model

The model reads every skill's description and starts a skill when the task matches. The description is therefore a trigger: it should name the situations, in the words a person would use.

| Weak description | Strong description |
|---|---|
| Helps with releases. | Draft release notes from merged changes since the last tag. Use when asked for release notes or a changelog. |
| Bug tool. | Diagnose and fix a reproducible defect: reproduce, find the root cause, freeze a failing test, fix, verify. |

### By the person

Typing `/name` starts the skill directly. Text after the name is passed as arguments.

```text
> /release-notes v2.3.0
```

| In the body | Becomes |
|---|---|
| `$ARGUMENTS` | all the arguments, `v2.3.0` here |
| `$ARGUMENTS[0]` or `$0` | the first argument |
| `$since` | a named argument declared in `arguments` |

If the body has no placeholder, Claude Code appends the arguments as `ARGUMENTS: <value>`.

- Skills with side effects, such as deploying or sending a message, are often set with `disable-model-invocation: true`, so only the person can start them.
- A skill the person starts still runs under the same permission rules and hooks as any other work.

## Writing the body

### Numbered steps

A skill body is a numbered procedure. Each step says what to do, what to check, and what to do when the check fails.

```markdown
1. List merged changes: `git log --merges --oneline <last-tag>..HEAD`.
2. Group them by area using the labels in `reference.md`.
3. For each group, write one sentence per user-visible change. Skip internal refactors.
4. Check every change from step 1 appears or is marked internal. If one is missing, add it.
5. Write the result to `RELEASE_NOTES.md` and show the diff.
```

### Put scripts where precision matters

The model is good at judgment and inconsistent at doing the same fiddly thing identically every time. A step that must be exact runs a script shipped with the skill.

| Step | As an instruction | As a script |
|---|---|---|
| Parse the last version tag | the model reads `git tag` output and picks one | `latest_tag.py` returns one tag or fails |
| Validate the output file | "check the file is well formed" | `validate.py notes.md` exits non-zero with the line number |
| Compute what changed | the model reads a long log | `changes.py <tag>` prints a fixed table |

This is the [systems approach]({{ '/sections/02-ai-harnesses/' | relative_url }}#a-systems-approach-to-harness-engineering) inside one skill: the workflow carries the enforcement, and the text explains it.

### Point at detail

Keep the body short enough to read in one pass.

- Steps and their checks stay in `SKILL.md`.
- Tables, edge cases and long examples move to reference files.
- Each reference file is named at the step that needs it ("For the label list, read `reference.md`"), so it loads only then.

## Worked example

### The prompt you kept retyping

"There's a bug where X. Find out why, write a test that fails, fix it, make sure everything passes, and tell me what the cause was."

- Typed by hand each time, with small differences.
- Some step went missing almost every time, most often the failing test.
- The report came back in a different shape each time.

### As a skill

```markdown
---
name: bug-fix
description: Diagnose and fix a reproducible defect. Reproduce, find the root cause, freeze a failing regression test, fix, verify.
---

1. Reproduce with the smallest command that shows the bug. Save it in the report.
2. Find the root cause. Name the line and why it is wrong. A guess is not a cause.
3. Write a test that fails for that reason. Commit the test alone.
4. Fix until the test passes. Run the full suite and fix every failure.
5. Report the cause, the test and the fix.
```

### What changed

| Before | After |
|---|---|
| test often skipped | test is step 3, before the fix, committed on its own |
| "find out why" accepted a guess | step 2 asks for the line and why it is wrong |
| report shape varied | step 5 names the three facts to report |
| prompt retyped | `/bug-fix <description>` |

## Skills in my harness

My harness has skills for the procedures I run most: landing a change end to end, fixing a bug, delegating work to another agent with a full brief, reviewing a change with a model from another family, saving session state before compaction, and drafting messages in my voice. Each began as a prompt I typed more than twice. A path rule with skill-authoring standards loads whenever a `SKILL.md` is edited, so new skills follow the same shape.

<details markdown="1">
<summary>Testing a skill</summary>

Run the skill on a real task in a fresh session and read the transcript. Check that it triggered from a natural request, that each step ran in order, and that the report has every field. When a step is skipped, the fix is usually in the step's wording or a missing check.

</details>

## Skills, commands and plugins

Claude Code has merged custom commands into skills. A file at `.claude/commands/deploy.md` and a skill folder named `deploy` under `.claude/skills/` both create `/deploy`; old command files keep working, and on a name clash the skill wins.

| | Skill | Command file | Plugin |
|---|---|---|---|
| What it is | a folder with `SKILL.md` and supporting files | one Markdown file | a package of skills, agents, hooks and MCP servers |
| Started by | the model or `/name` | `/name`, or the model | installs the parts it contains |
| Best for | multi-step procedures with scripts and references | a short reusable prompt | sharing a set with others |

## Pitfalls

| Pitfall | Result | Fix |
|---|---|---|
| a vague description | the skill never starts on its own | name the situations and the words a person uses |
| a description that matches too much | the skill starts on unrelated tasks | narrow it, or set `paths` |
| a skill with side effects the model can start | it runs when you did not mean it to | `disable-model-invocation: true` |
| a long body | it stays in the conversation after use | move detail to reference files |
| steps the model must get exactly right | skipped or done differently each time | a script, run by the step |

<!-- IMAGE NEEDED: the folder of one skill (SKILL.md, reference file, script) next to the slash-command menu where it appears; search: claude code skills folder structure, agent skills SKILL.md diagram, agentskills.io -->

See [Plugins and MCP servers]({{ '/sections/02-ai-harnesses/plugins-and-mcp-servers.html' | relative_url }}).

## Sources

- Claude Code skills: <https://code.claude.com/docs/en/skills.md>, verified 2026-11-09.
- Codex skills: <https://learn.chatgpt.com/docs/build-skills>, verified 2026-11-09.
- Agent Skills format: <https://agentskills.io/>.
- The skill examples are from the author's own harness, described conceptually.

{% include page_nav.html %}
