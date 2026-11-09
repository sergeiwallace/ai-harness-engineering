---
title: "Skills, commands and plugins"
nav_order: 15
has_children: true
has_toc: false
permalink: /sections/05-skills-and-plugins/
---

{% include page_nav.html %}

# Skills, commands and plugins

As of 2026-11-09.

Stop 4 on the talk path, about 1.5 minutes.

A skill is a written procedure the agent loads only when it needs it, and runs the same way each time. A plugin packages skills with sub-agents, hooks and MCP servers so a team installs them as one unit. A mod is a plugin that changes how the client itself looks and behaves.

![Sequence diagram with five lanes: Person, Client, Context window, Model and Skill folder. At session start the client puts the names and descriptions of every skill into the context window. The person types /release 2.4.0; the client reads SKILL.md and its body enters the window; the model starts step 1. At step 2 the model reads checklist.md and its contents enter the window. At step 3 the model runs bump_version.py and only the script output enters the window. The model reports to the person that the tag is ready.]({{ '/assets/images/05-skills-and-plugins/skill-loading-sequence.png' | relative_url }})
*A release skill during a real task. The window holds every skill's description from the start; the body, the checklist and the script's output arrive only at the step that needs them.*

This overview covers every part of the topic with a worked example for each. Three topic pages go deeper:

- [Skill anatomy and loading]({{ '/sections/05-skills-and-plugins/skill-anatomy.html' | relative_url }}): the folder, every frontmatter field, how the three loading layers work, arguments and injected command output, and the Agent Skills standard. Its sub-page [Writing skill descriptions]({{ '/sections/05-skills-and-plugins/skill-descriptions.html' | relative_url }}) covers how the model picks a skill.
- [Invocation and chaining]({{ '/sections/05-skills-and-plugins/invocation-and-chaining.html' | relative_url }}): who may start a skill, the trap that breaks chains, and building pipelines out of skills.
- [Plugins, marketplaces and mods]({{ '/sections/05-skills-and-plugins/plugins-and-mods.html' | relative_url }}): packaging parts as one unit, hosting a marketplace, the context cost, mods, and a shared code library across skills.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Where skills fit in a harness

A harness has several places to put an instruction. A context file (`CLAUDE.md`, `AGENTS.md`) loads on every turn, so it suits short rules that always apply. A hook runs code at a fixed event, so it suits rules that must hold every time. A skill sits between them: a written procedure that loads only when it is used, and runs the same steps each time it does.

| Part | Loads | Enforced | Best for |
|---|---|---|---|
| Context file | Every turn | No | Short rules that always apply |
| Skill | Description always, body on use | Steps are fixed once it runs | Long procedures needed sometimes |
| Hook | Never in the window; runs as code | Yes | Rules that must hold every time |
| Plugin | Whatever its parts load | As its parts are | Sharing a set of parts as one unit |

### Example: placing one procedure

A team has a 40-line release procedure. In the context file it would cost 40 lines on every turn of every session, including the sessions that never release. As a skill it costs one line, the description, until someone types `/release`. The one rule inside it that must never be broken, "never tag a release from a branch other than main", also goes into a hook, because a skill only helps when it runs.

#### The split, line by line

| Line of the old procedure | Goes to | Why |
|---|---|---|
| "Releases are cut on Tuesdays" | Context file | Short, always true, no program can check intent |
| Steps 1 to 12 of the release | Skill body | Long, needed only when releasing |
| The changelog template | File in the skill folder | Needed at one step only |
| "Never tag from a branch other than main" | Hook on `git tag` | Must hold every time, checkable by code |

> A skill is followed when it runs, and nothing forces it to run. A rule that must always hold needs a hook as well. Section 6 covers [the enforcement ladder]({{ '/sections/06-hooks-scripts-and-linters/enforcement-ladder.html' | relative_url }}).

## What a skill is

A skill is a folder with a `SKILL.md` file of instructions and, optionally, scripts, templates and reference files. In Claude Code, custom slash commands have been merged into skills: a skill named `release` is started with `/release`, and an old `.claude/commands/release.md` file keeps working. Skills follow the Agent Skills open standard, which Codex also uses.

```text
.claude/skills/<name>/    # here <name> is release
├── SKILL.md           # frontmatter + steps
├── checklist.md       # read at step 2
└── bump_version.py    # run at step 3
```

Skills live at three levels: the user level (`~/.claude/skills/`) for your own procedures in every project, the project level (`.claude/skills/`) for procedures the whole team shares through git, and inside a plugin. The [anatomy page]({{ '/sections/05-skills-and-plugins/skill-anatomy.html' | relative_url }}#the-folder) has the full layout.

### Example: a small skill

```markdown
---
name: bug-report
description: Write a bug report from a failing test or a pasted error. Use when the user says "file a bug" or "write this up".
---
1. Reproduce the failure and record the exact command and output.
2. Find the smallest input that still fails.
3. Write the report with: summary, steps, expected, actual, environment.
4. Save it to docs/bugs/<date>-<slug>.md and show the path.
```

You type `/bug-report` with the error, or the model picks the skill itself when you say "file this as a bug". Either way it runs the same four steps, and every report on the team has the same five headings.

#### What a session looks like

```text
> /bug-report TypeError: cannot read 'id' of undefined in checkout.test.ts

  Running the bug-report skill.
  1. Reproduced: npm test -- checkout.test.ts fails at line 88.
  2. Smallest failing input: a cart with one item and no customer.
  3. Report written with summary, steps, expected, actual, environment.
  4. Saved: docs/bugs/2026-11-09-checkout-missing-customer.md
```

<details markdown="1">
<summary>Skills and the old custom commands</summary>

Before skills, Claude Code had custom slash commands: one Markdown file per command in `.claude/commands/`. Those files still work, and a command and a skill with the same name start the same way. A skill adds what a single file cannot: a folder for scripts and reference files, frontmatter that controls who can start it, and loading of its parts on demand. New procedures go in skills.

</details>

## Built-in commands and skills

Everything typed after a `/` in Claude Code is either a built-in command or a skill. Built-in commands ship with the client and act on the session itself: its context, its permissions, its plugins. Skills are procedures you or a plugin add. Both appear in the same menu when you type `/`.

| | Built-in command | Skill |
|---|---|---|
| Comes from | The client | You, your team or a plugin |
| Acts on | The session: context, settings, plugins | The work: code, documents, releases |
| Who starts it | Usually you, by typing it | You or the model, unless person-only |
| Can you change it | No | Yes, it is a file you edit |

### Four commands the talk uses

Checked against Claude Code v2.1.319. Several commands changed meaning in recent releases, so check the commands reference for your version.

| Command | What it does | Worked example |
|---|---|---|
| `/goal` | Keeps the session working until a condition is met | `/goal all tests in tests/api pass and the PR is open` keeps the agent going past the first green run |
| `/loop` | Repeats a prompt on an interval | `/loop 10m check the deploy and report any failed job` polls without you asking each time |
| `/compact` | Summarises the session to free context; its focus text steers the summary | `/compact keep the open decisions and the failing test names` before a long second half |
| `/rc` | Controls a session from another device; needs a claude.ai sign-in | Start a long run at the desk, then answer its question from a phone |

### Commands for managing skills and plugins

| Command | Use it to |
|---|---|
| `/skills` | List the skills this session can see |
| `/reload-skills` | Pick up an edited `SKILL.md` without a restart |
| `/skill-doctor` | See what each skill costs in context and how often it runs |
| `/plugin` | List, install, enable and disable plugins |
| `/reload-plugins` | Apply plugin changes without a restart |

#### Example: editing a skill mid-session

```text
> /pr-description
  (writes a description without the Rollout notes heading)

  You edit the skill's SKILL.md to add the heading.

> /reload-skills
  Reloaded 14 skills.

> /pr-description
  (the description now ends with Rollout notes)
```

The full list of built-in commands, with the ones each section of this site uses, is on the [Claude Code and Codex overview]({{ '/sections/03-claude-code-and-codex/' | relative_url }}).

## How a skill loads

![Three stacked layers. Layer 1, the name and description of every skill, is always loaded. Layer 2, the SKILL.md body, loads on use. Layer 3, files in the skill folder, load on demand.]({{ '/assets/images/05-skills-and-plugins/skill-loading-layers.svg' | relative_url }})

A skill loads in three layers:

1. The name and description of every model-invocable skill sit in the context window on every turn.
   - The model uses them to know the skill exists and to choose it.
2. The body loads when the skill is used.
3. Files in the folder load only when a step reads or runs them.
   - A script adds only its output.

A large set of skills therefore costs little until one runs, as long as the descriptions stay short.

### Example: what the window holds

A project has 30 skills. Before anyone uses one, the window holds 30 descriptions of about 25 words each. When the person types `/release`, the 60-line body joins them. At step 2 the agent reads `checklist.md`, and only then do its 80 lines load. The other 29 bodies never load in that session.

The [anatomy page]({{ '/sections/05-skills-and-plugins/skill-anatomy.html' | relative_url }}#how-loading-works) shows the sequence as a diagram and covers the frontmatter fields that change it, such as running a skill in a forked context.

## Arguments and live data

A skill can take arguments. Whatever you type after `/name` is available to the body as `$ARGUMENTS`, so `/fix-issue 412` can tell the agent which issue to fix. A skill body can also run a shell command at load time and paste its output into the instructions, with the `` !`command` `` syntax, so the agent starts with live data and does not have to fetch it.

### Example: a skill that arrives with its data

```markdown
---
name: pr-description
description: Write a pull request description for the current branch. Use before opening a pull request.
---
Commits on this branch:
!`git log --oneline main..HEAD`

Files changed:
!`git diff --stat main...HEAD`

Write: Summary (two sentences), Linked issue, Changes (one line per file group),
Test evidence (the exact test command and its last line of output), Rollout notes.
Never claim a test passed unless its output is in this session.
```

The two git commands run when the skill loads, so the agent sees the commits and files before it writes a word. The last line came from an agent that reported passing tests it had not run. It is the second layer; the enforcing check is a CI job that fails a pull request whose test evidence section has no command output.

## Who can start a skill

By default both you and the model can start a skill. Two frontmatter settings change that:

| Frontmatter | You can start it | The model can start it | Use for |
|---|---|---|---|
| (default) | Yes | Yes | Procedures the model should use whenever they fit |
| `disable-model-invocation: true` | Yes | No | Side effects you must choose: publish, send, merge, delete |
| `user-invocable: false` | No | Yes | Background procedures that should stay out of the command menu |

Codex has a similar setting, `allow_implicit_invocation: false`.

### Example: two skills, two settings

`/run-tests` runs the project's test procedure; the model starts it after every change, so it keeps the default. `/publish-release` pushes a tag that customers download; only a person decides to ship, so it sets `disable-model-invocation: true`.

> A person-only skill cannot be started by another skill, a hook or an agent's brief. If something in the harness calls it, the chain stops without an error. Keep such a skill model-invocable and put the gate inside it.

The [invocation page]({{ '/sections/05-skills-and-plugins/invocation-and-chaining.html' | relative_url }}#who-can-start-a-skill) has a decision table and the one trap this setting causes.

## Writing descriptions

The model picks a skill by matching the request against the descriptions. A good description says what the skill does, when to use it in the words a person would say, and when to use a neighbouring skill instead. A vague description means the skill is never picked; two overlapping descriptions mean the wrong one is.

### Example: fixing an overlap

| Skill | Before | After |
|---|---|---|
| `review` | Review code | Review the current diff before a commit or pull request |
| `audit` | Audit code for problems | Audit a whole module for design problems. Use only when the user asks for an audit |

Before the change the model picked between them at random. After it, a request to "check my changes" goes to `review`, and only an explicit audit request goes to `audit`. The [sub-page on descriptions]({{ '/sections/05-skills-and-plugins/skill-descriptions.html' | relative_url }}) has a checklist and more before-and-after pairs.

## Chaining skills

Skills combine in two ways. You can start one message with several skills, up to six, and Claude Code loads them all before the task. And a skill's instructions can tell the agent to run another skill at a step, which builds a pipeline out of small parts. The second form is a convention: it works because the agent reads "now run `/review`" as an instruction.

![Sequence diagram with five lanes: Person, Model, /run-tests, /review and /pr-description. The person types /ship. Step 1: the model runs /run-tests and 48 pass; the chain would stop on a failure. Step 2: the model runs /review on the diff, which returns review.md with one high and two low findings; the model fixes the high finding and runs /run-tests again, and 48 pass. Step 3: the model runs /pr-description, gets the summary, tests and rollout notes, and reports that pull request 88 is open.]({{ '/assets/images/05-skills-and-plugins/ship-chain-sequence.png' | relative_url }})
*A `/ship` skill during a real run. Each called skill returns a named output that the next step reads, and the test and review steps can stop the chain.*

### Example: one launch path

My orchestration skill plans a piece of work and hands each launch to my delegation skill, which writes the brief and starts the worker in its own checkout. Because every launch goes through one skill, a fix to how briefs are written lands once and applies to every launch. The [chaining part of the invocation page]({{ '/sections/05-skills-and-plugins/invocation-and-chaining.html' | relative_url }}#chaining-skills) covers the patterns and how chains fail.

## Plugins and marketplaces

A plugin is "a directory of skills, agents, hooks, MCP servers, or other components that Claude Code installs and loads as one unit". A marketplace is a catalogue of plugins, often a git repository, that people add once and install from. `/plugin` lists, installs, enables and disables plugins, and `/reload-plugins` applies changes without a restart.

```text
review-kit/
├── .claude-plugin/plugin.json      # name, version, description
├── skills/<name>/SKILL.md      # here <name> is review
├── agents/security-reviewer.md
├── hooks/hooks.json                # lint before each commit
└── .mcp.json
```

### Example: a review plugin

A team packages its review skill, a sub-agent profile for a security reviewer, and a hook that runs the linter before each commit as one plugin, `review-kit`, in the team's marketplace repository. A new team member adds the marketplace once and installs `review-kit` with one command, and has all three parts at the same version as everyone else.

### The cost

Every enabled plugin costs context on every turn, because the name and description of each model-invocable skill and agent in it stay loaded. `/skill-doctor` shows what each skill costs and how often it runs. Install what you use and disable the rest. The [plugins page]({{ '/sections/05-skills-and-plugins/plugins-and-mods.html' | relative_url }}) covers the manifest, hosting a marketplace and the cost in more depth.

## Mods

A mod is a plugin with a hooks module: hooks written as JavaScript or TypeScript functions that change how Claude Code looks and behaves, such as drawing a pane in the terminal beside the chat. Mods need Claude Code 2.1.287 or later.

### Example: a context pane

A mod draws a small pane that shows how full the context window is and refreshes after each turn. The person sees the number without asking for it, and decides when to compact. My harness has mods of this kind.

## Shared code across skills

Skills often include small command-line tools. Written one per skill, each parses options its own way, loads settings its own way and has its own test helpers, and a bug fixed in one stays in the others. A plugin whose skills, MCP servers and hooks all import one shared library avoids that. I call this a plugin module; that is my own name, and Claude Code has no such term.

![Sequence diagram with five lanes: Developer, Shared library, Skill A tool, Skill B tool and MCP server. Two skill tools call settings.load and both get a wrong path on a second profile. The developer fixes the profile lookup in the library once and adds a test, which passes. After the next install, Skill A, Skill B and the MCP server each call settings.load and get the correct path, with no change to any of them.]({{ '/assets/images/05-skills-and-plugins/shared-library-fix-sequence.png' | relative_url }})
*A settings bug fixed once in the shared library. Every tool, hook and server that imports it gets the fix on the next install.*

### Example: one fix, every tool

A bug in how tools read a settings file affected every skill tool that read settings. With the shared library the fix was one change in the settings module, and every tool picked it up on the next install. The [plugins page]({{ '/sections/05-skills-and-plugins/plugins-and-mods.html' | relative_url }}#shared-code-across-skills) shows the module layout.

## Keeping a growing set useful

Skills pile up. Prune the ones nobody invokes, test the parts that can be tested, and enforce a rule that must always hold with a hook or a test, because a skill body loads only on use.

### Example: a monthly prune

Once a month, list the skills used in the last 30 days. A skill nobody ran is deleted, or merged into a neighbour if its steps are still useful. Two skills that are always run together become one. A test checks that every `SKILL.md` has valid frontmatter and that no person-only skill is called by another skill.

#### A frontmatter test

```python
import pathlib, re, yaml

SKILLS = pathlib.Path(".claude/skills")

def frontmatter(path):
    text = path.read_text()
    return yaml.safe_load(text.split("---")[1])

def test_every_skill_has_name_and_description():
    for f in SKILLS.glob("*/SKILL.md"):
        meta = frontmatter(f)
        assert meta.get("name") == f.parent.name, f
        assert 20 <= len(meta.get("description", "")) <= 300, f

def test_no_skill_calls_a_person_only_skill():
    person_only = {f.parent.name for f in SKILLS.glob("*/SKILL.md")
                   if frontmatter(f).get("disable-model-invocation")}
    for f in SKILLS.glob("*/SKILL.md"):
        body = f.read_text().split("---", 2)[2]
        for name in person_only - {f.parent.name}:
            assert not re.search(rf"/{name}\b", body), f"{f} calls /{name}"
```

## Examples from my own harness

- Person-only skills for actions with side effects, such as pulling from git or filing product feedback. A skill that another skill or a hook calls stays model-invocable, or the chain breaks.
- Chained skills: the orchestration skill hands its launch step to the delegation skill.
- A shared library for the command-line tools inside skills.
- A core plugin that serves the same read-only tools to Claude Code and to Codex, so both clients query the same knowledge store the same way.
- Mods that draw status panes inside the terminal session, such as a pane showing how full the context window is.

To start: turn the prompt you type most often into a skill.

## Pitfalls and lessons

- Skills with overlapping descriptions make the model pick the wrong one.
- A person-only skill called by another part of the harness breaks the chain without an error.
- Every enabled plugin costs context. Disable what you do not use.
- A rule that must always hold needs a hook or a test; a line in a skill body is followed only when the skill runs.

## Sources

- Skills, the file format, arguments, injected command output and invocation settings: <https://code.claude.com/docs/en/skills.md>, verified 2026-11-09.
- Several skills in one message, up to six; `/plugin`, `/reload-plugins`, `/skill-doctor`: <https://code.claude.com/docs/en/commands.md>, verified 2026-11-09.
- Plugins, marketplaces, the hooks module and mods: <https://code.claude.com/docs/en/plugins/overview.md> and <https://code.claude.com/docs/en/plugins/mods/overview.md>, verified 2026-11-08.
- Codex skills: <https://learn.chatgpt.com/docs/build-skills.md>, verified 2026-11-08.
- The shared library, chained launch path, core plugin and mods are from the author's own harness, described conceptually. The example skills and plugin are illustrations written for this page. The loading-layers diagram was drawn for this page.

{% include page_nav.html %}
