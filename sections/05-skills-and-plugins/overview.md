---
title: "Skills, commands and plugins"
nav_order: 15
has_children: true
has_toc: false
permalink: /sections/05-skills-and-plugins/
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Skills, commands and plugins

Stop 4 on the talk path, about 1.5 minutes.

A skill is a written procedure that the agent loads when it needs it and runs the same way each time. A built-in command is part of the client and acts on the session itself. A plugin packages skills with sub-agent profiles, hooks and MCP servers so a team installs them as one unit, and a mod is a plugin whose function hooks run inside the client on events such as tool calls and prompts.

![Sequence diagram with five lanes: Person, Client, Context window, Model and Skill folder. At session start the client puts the names and descriptions of every skill into the context window. The person types /release 2.4.0; the client reads SKILL.md and its body enters the window; the model starts step 1. At step 2 the model reads checklist.md and its contents enter the window. At step 3 the model runs bump_version.py and only the script output enters the window. The model reports to the person that the tag is ready.]({{ '/assets/images/05-skills-and-plugins/skill-loading-sequence.png' | relative_url }})
*A release skill during a real task. The window holds every skill's description from the start; the body, the checklist and the script's output arrive only at the step that needs them.*

## How this section is organised

This overview covers each part with a worked example, the settings and commands involved, and the edge cases to watch. Three topic pages go deeper:

| Page | Covers |
|---|---|
| [Skill anatomy and loading]({{ '/sections/05-skills-and-plugins/skill-anatomy.html' | relative_url }}) | The folder, every frontmatter field, the three loading layers, forked context, arguments, injected command output and the Agent Skills standard |
| [Writing skill descriptions]({{ '/sections/05-skills-and-plugins/skill-descriptions.html' | relative_url }}) (sub-page of the anatomy page) | How the model picks a skill, a description template, before-and-after pairs and a checklist |
| [Invocation and chaining]({{ '/sections/05-skills-and-plugins/invocation-and-chaining.html' | relative_url }}) | Who may start a skill, the trap that stops chains, pipeline patterns and how chains fail |
| [Plugins, marketplaces and mods]({{ '/sections/05-skills-and-plugins/plugins-and-mods.html' | relative_url }}) | Plugin layout and manifest, hosting a marketplace, the context cost and a shared code library |
| [Mods and function hooks]({{ '/sections/05-skills-and-plugins/mods-and-function-hooks.html' | relative_url }}) | Function hooks inside the client: events, the comparison with shell hooks, behaviour and interface mods, building and debugging |

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## What a skill is

A skill is a folder with a `SKILL.md` file of instructions and, optionally, scripts, templates and reference files. The [anatomy page]({{ '/sections/05-skills-and-plugins/skill-anatomy.html' | relative_url }}) takes one apart.

- `SKILL.md` holds YAML frontmatter (a name, a description and optional settings) and then the steps in Markdown.
- In Claude Code, custom slash commands have been merged into skills. A skill named `release` starts with `/release`, and an old `.claude/commands/release.md` file keeps working.
- Skills follow the Agent Skills open standard, which Codex also reads.

```text
.claude/skills/<name>/    # here <name> is release
├── SKILL.md           # frontmatter + steps
├── checklist.md       # read at step 2
└── bump_version.py    # run at step 3
```

### Where skill folders live

| Level | Folder | Shared with |
|---|---|---|
| User | `~/.claude/skills/<name>/` | Only you, in every project |
| Project | `.claude/skills/<name>/` | Everyone who clones the repository |
| Plugin | `<plugin>/skills/<name>/` | Everyone who installs the plugin |
| Codex project | `.agents/skills/<name>/` | Codex sessions in the repository |

Give skills at different levels different names. When two levels define the same name, the client picks one by the precedence order in its docs.

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

## Where skills fit in a harness

A harness enforces its rules in four layers, described in the [AI harnesses overview]({{ '/sections/02-what-is-a-harness/' | relative_url }}): instructions; workflows and tools; hooks, scripts and linters; validation and tests. A skill spans the first two.

- Its numbered steps and the scripts it runs build a procedure into the work (layer 2).
- Its prose is still an instruction the model reads (layer 1), followed as unevenly as any context-file line.
- It holds only when it runs. A rule that must always hold also needs a hook or a test.

| Part | Loads | Enforces | Best for |
|---|---|---|---|
| Context file | Every turn | No | Short rules that always apply |
| Skill | Description always, body on use | Its steps and scripts, once it runs | Long procedures needed sometimes |
| Hook | Never in the window; runs as code | Yes | Rules that must hold every time |
| Plugin | Whatever its parts load | As its parts do | Sharing a set of parts as one unit |

### Example: placing one procedure

A team has a 40-line release procedure. In the context file it would cost 40 lines on every turn of every session, including the sessions that never release. Split by where each line belongs:

| Line of the old procedure | Goes to | Why |
|---|---|---|
| "Releases are cut on Tuesdays" | Context file | Short and always true; no program can check intent |
| Steps 1 to 12 of the release | Skill body | Long, and needed only when releasing |
| The changelog template | File in the skill folder | Needed at one step only |
| "Check the version matches in three files" | Script the skill runs | A program can do it exactly |
| "Never tag from a branch other than main" | Hook on `git tag` | Must hold every time, and code can check it |

## How a skill loads

![Three stacked layers. Layer 1, the name and description of every skill, is always loaded. Layer 2, the SKILL.md body, loads on use. Layer 3, files in the skill folder, load on demand.]({{ '/assets/images/05-skills-and-plugins/skill-loading-layers.svg' | relative_url }})

A skill loads in three layers:

1. The name and description of every model-invocable skill sit in the context window on every turn.
   - The model uses them to know the skill exists and to choose it.
2. The body loads when the skill is used.
3. Files in the folder load only when a step reads or runs them.
   - A script adds only its output.

A large set of skills costs little until one runs, as long as the descriptions stay short.

### Example: what the window holds

A project has 30 skills. The window grows in steps during one release:

| Moment | Added to the window | Running total |
|---|---|---|
| Session start | 30 descriptions of about 25 words | about 750 words |
| Person types `/release` | The 60-line body | plus 60 lines |
| Step 2 reads `checklist.md` | Its 80 lines | plus 80 lines |
| Step 3 runs `bump_version.py` | One line of output | plus 1 line |

The other 29 bodies never load in that session.

### Edge cases

- A person-only skill (`disable-model-invocation: true`) keeps its description out of the window too, so it costs nothing until someone types it.
- A skill with `context: fork` runs in its own sub-agent context; its reading stays there and only the result returns. The [anatomy page]({{ '/sections/05-skills-and-plugins/skill-anatomy.html' | relative_url }}#running-in-a-forked-context) covers when that helps.
- An edited `SKILL.md` is picked up with `/reload-skills`; until then the session uses the old version.

## Writing descriptions

The model picks a skill by matching the request against the descriptions, before it has read any body. A good description has three parts:

1. What the skill does, with a verb and an object: "Write a pull request description for the current branch."
2. When to use it, in the words a person would say: "Use before opening a pull request."
3. When to use a neighbouring skill, if one is close: "For a whole module, use audit."

A vague description is rarely picked. Two descriptions that overlap lead the model to pick the wrong one.

### Example: fixing an overlap

| Skill | Before | After |
|---|---|---|
| `review` | Review code | Review the current diff before a commit or pull request |
| `audit` | Audit code for problems | Audit a whole module for design problems. Use only when the user asks for an audit |

![Sequence diagram with four lanes: Person, Model, review and audit. Before the rewrite, a request to check the change before a push goes to audit, which reads the whole module for several minutes. After the rewrite, in a new session, the same request goes to review and returns three findings in a minute; audit is skipped.]({{ '/assets/images/05-skills-and-plugins/skill-pick-sequence.png' | relative_url }})
*The same request before and after the rewrite. Test a description change in a new session, where the new text is the one in the window.*

The [sub-page on descriptions]({{ '/sections/05-skills-and-plugins/skill-descriptions.html' | relative_url }}) has a template, more before-and-after pairs and a checklist.

## Arguments and live data

A skill can receive data in two ways:

- Arguments: whatever you type after `/name` is available to the body as `$ARGUMENTS`.
  - `/fix-issue 412` turns a step `gh issue view $ARGUMENTS` into `gh issue view 412`.
  - If nothing follows the command, `$ARGUMENTS` is empty, so a step that needs it should say what to do then.
- Injected command output: a line of the form `` !`command` `` runs when the skill loads, and its output replaces the line.
  - The agent starts with the data in front of it and spends no tool calls fetching it.
  - The command runs with your permissions, so read a skill from someone else before installing it.

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

- The two git commands run when the skill loads, so the agent sees the commits and files before it writes.
- The last line came from an agent that reported passing tests it had not run. It is the second layer; the enforcing check is a CI job that fails a pull request whose test evidence section has no command output.

## Who can start a skill

By default both you and the model can start a skill. Two frontmatter settings change that:

| Frontmatter | You can start it | The model can start it | Use for |
|---|---|---|---|
| (default) | Yes | Yes | Procedures the model should use whenever they fit |
| `disable-model-invocation: true` | Yes | No | Side effects you must choose: publish, send, merge, delete |
| `user-invocable: false` | No | Yes | Background procedures that stay out of the command menu |

Codex has a similar setting, `allow_implicit_invocation: false`.

### Example: two skills, two settings

| Skill | What it does | Setting | Reason |
|---|---|---|---|
| `/run-tests` | Runs the project's test procedure | Default | The model should run it after every change |
| `/publish-release` | Pushes a tag that customers download | `disable-model-invocation: true` | Only a person decides to ship |

### Edge case: the chain trap

A person-only skill cannot be started by another skill, a hook's message or an agent's brief, because all of those go through the model.

![Sequence diagram with four lanes: Person, Model, Client and /publish-release, which is person-only. The model runs steps 1 to 3, then asks to run /publish-release; the client answers that it is not available; the model reports release day complete although step 4 never ran. After the fix the skill is model-invocable with a confirm step, so the model runs it and the person approves.]({{ '/assets/images/05-skills-and-plugins/chain-trap-sequence.png' | relative_url }})
*The chain stops without an error, and the model reports the run as complete.*

> Keep a skill model-invocable if anything in the harness calls it, and put the gate on the action itself: a confirmation step in the skill, or a hook on the dangerous command. The [invocation page]({{ '/sections/05-skills-and-plugins/invocation-and-chaining.html' | relative_url }}#the-chain-trap) has the full case and a test that finds it.

## Chaining skills

Skills combine in two ways:

1. A person starts one message with several skills, up to six: `/review /pr-description get this ready to merge` loads both before the task.
2. A skill's step tells the agent to run another skill, which builds a pipeline out of small parts.
   - This is a convention; the docs do not describe it.
   - It works because the agent reads "now run `/review`" as an instruction.

![Sequence diagram with five lanes: Person, Model, /run-tests, /review and /pr-description. The person types /ship. Step 1: the model runs /run-tests and 48 pass; the chain would stop on a failure. Step 2: the model runs /review on the diff, which returns review.md with one high and two low findings; the model fixes the high finding and runs /run-tests again, and 48 pass. Step 3: the model runs /pr-description, gets the summary, tests and rollout notes, and reports that pull request 88 is open.]({{ '/assets/images/05-skills-and-plugins/ship-chain-sequence.png' | relative_url }})
*A `/ship` skill during a real run. Each called skill returns a named output that the next step reads, and the test and review steps can stop the chain.*

### Example: one launch path

In my harness one chain handles every delegated task:

1. The orchestration skill plans the work and splits it into tasks.
2. For each task it runs the delegation skill.
   - The delegation skill writes the brief from one template.
   - It starts the worker in its own checkout.
3. Each result is reviewed and merged.

Every launch goes through the one delegation skill, so a fix to how briefs are written applies to every launch.

### Edge cases

- A long chain fills the window, and later steps lose track of early instructions. Fork the heavy steps or keep the chain short.
- A gate that lives only inside a skill is skipped whenever the skill is. Put must-hold gates in a hook or a test as well.

The [chaining part of the invocation page]({{ '/sections/05-skills-and-plugins/invocation-and-chaining.html' | relative_url }}#chaining-skills) covers the patterns and how chains fail.

## Built-in commands and skills

Everything typed after a `/` in Claude Code is either a built-in command or a skill, and both appear in the same menu.

| | Built-in command | Skill |
|---|---|---|
| Comes from | The client | You, your team or a plugin |
| Acts on | The session: context, settings, plugins | The work: code, documents, releases |
| Who starts it | Usually you, by typing it | You or the model, unless person-only |
| Can you change it | No | Yes, it is a file you edit |

<!-- IMAGE NEEDED: the Claude Code slash-command menu open after typing "/", showing built-in commands and project skills in one list; search: claude code slash command menu, claude code skills menu screenshot -->

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

## Plugins and marketplaces

A plugin is "a directory of skills, agents, hooks, MCP servers, or other components that Claude Code installs and loads as one unit".

- Its manifest, `.claude-plugin/plugin.json`, holds the name, version and description.
- A marketplace is a catalogue of plugins, often a git repository, that people add once and install from.
- `/plugin` lists, installs, enables and disables plugins; `/reload-plugins` applies changes without a restart.

```text
review-kit/
├── .claude-plugin/plugin.json      # name, version, description
├── skills/<name>/SKILL.md      # here <name> is review
├── agents/security-reviewer.md
├── hooks/hooks.json                # lint before each commit
└── .mcp.json
```

### Example: a review plugin

A team packages three parts as one plugin, `review-kit`, in its marketplace repository:

- A review skill for the current diff.
- A sub-agent profile for a security reviewer.
- A hook that runs the linter before each commit.

A new team member installs it in two commands:

```text
> /plugin marketplace add example-org/platform-tools
> /plugin install review-kit@platform-tools
  Installed review-kit 1.4.0: 1 skill, 1 agent, 1 hook.
```

![Sequence diagram with four lanes: New team member, Client, Marketplace repo and Context window. The person adds the marketplace and installs review-kit; the client fetches version 1.4.0 with one skill, one agent profile and one hook. From then on the review skill and reviewer descriptions enter the context window on every turn, and the lint hook runs before each commit.]({{ '/assets/images/05-skills-and-plugins/plugin-install-sequence.png' | relative_url }})
*Installing from a team marketplace, and what the plugin costs on every turn afterwards.*

### The cost

Every enabled plugin costs context on every turn:

| Part of the plugin | Cost on every turn |
|---|---|
| Model-invocable skill | Its name and description |
| Sub-agent profile | Its name and description |
| MCP server | The name and schema of each tool |
| Hook that injects text | That text, each time the hook fires |

`/skill-doctor` shows what each skill costs and how often it runs. Install what you use and disable the rest.

### Edge cases

- A cloud session does not load local plugins.
- The same plugin format installs on claude.ai and in Cowork, where a different set of components loads.
- A plugin whose version never changes leaves people unsure which copy they run. Raise the version on every change you ship.

The [plugins page]({{ '/sections/05-skills-and-plugins/plugins-and-mods.html' | relative_url }}) covers the manifest, hosting a marketplace and auditing the cost.

## Mods

A mod is a plugin whose hooks module registers function hooks: JavaScript or TypeScript functions that Claude Code loads into its own process and calls on events such as tool calls, prompts and turns.

- A function hook can watch an event, change it, or answer it in Claude Code's place, such as refusing a command or adding context to a prompt.
- Changing the interface, with a pane, a band above the prompt, a toast or a new slash command, is one common use; a mod that only guards tool calls draws nothing.
- Mods need Claude Code v2.1.287 or later in the terminal.

[Mods and function hooks]({{ '/sections/05-skills-and-plugins/mods-and-function-hooks.html' | relative_url }}) covers the event contract, the comparison with shell hooks, building and debugging a mod, and the context indicator from my harness.

## Shared code across skills

Skills often include small command-line tools. Written one per skill, they diverge:

- Each parses options its own way and loads settings its own way.
- Each has its own test helpers.
- A bug fixed in one stays in the others.

A plugin whose skills, MCP servers and hooks all import one shared library avoids that. I call this a plugin module; that is my own name, and Claude Code has no such term.

![Sequence diagram with five lanes: Developer, Shared library, Skill A tool, Skill B tool and MCP server. Two skill tools call settings.load and both get a wrong path on a second profile. The developer fixes the profile lookup in the library once and adds a test, which passes. After the next install, Skill A, Skill B and the MCP server each call settings.load and get the correct path, with no change to any of them.]({{ '/assets/images/05-skills-and-plugins/shared-library-fix-sequence.png' | relative_url }})
*A settings bug fixed once in the shared library. Every tool, hook and server that imports it gets the fix on the next install.*

### Example: one fix, every tool

| | One copy per skill | Shared library |
|---|---|---|
| Settings bug found | In every tool that reads settings | In one module |
| Fix | One change per tool, and some are missed | One change and one test |
| Reaches the tools | When each is edited | On the next install |

The [plugins page]({{ '/sections/05-skills-and-plugins/plugins-and-mods.html' | relative_url }}#shared-code-across-skills) shows the module layout and a tool built on it.

## Keeping a growing set useful

Skills pile up. Three habits keep a set useful:

- Prune the ones nobody invokes, using `/skill-doctor` for usage.
- Test the parts that can be tested: scripts, frontmatter, and calls between skills.
- Enforce a rule that must always hold with a hook or a test, because a skill body loads only on use.

### Example: a monthly prune

1. List the skills used in the last 30 days.
2. Delete a skill nobody ran, or merge it into a neighbour if its steps are still useful.
3. Merge two skills that are always run together.
4. Run the frontmatter test below, which checks every `SKILL.md` and every call between skills.

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

- Person-only skills for actions with side effects, such as pulling from git or filing product feedback. A skill that another skill or a hook calls stays model-invocable, with its gate on the action.
- Chained skills: the orchestration skill hands its launch step to the delegation skill.
- A shared library for the command-line tools inside skills.
- A core plugin that serves the same read-only tools to Claude Code and to Codex, so both clients query the same knowledge store the same way.
- A mod that shows how full the context window is after each answer.

To start: turn the prompt you type most often into a skill.

## Pitfalls and lessons

- Skills with overlapping descriptions make the model pick the wrong one.
- A person-only skill called by another part of the harness stops the chain without an error.
- Every enabled plugin costs context. Disable what you do not use.
- A rule that must always hold needs a hook or a test; a line in a skill body is followed only when the skill runs.

## Sources

- Skills, the file format, arguments, injected command output and invocation settings: [Claude Code skills documentation](https://code.claude.com/docs/en/skills.md), verified 2026-11-09.
- Several skills in one message, up to six; `/plugin`, `/reload-plugins`, `/skill-doctor`: [Claude Code commands reference](https://code.claude.com/docs/en/commands.md), verified 2026-11-09.
- Plugins, marketplaces, the hooks module and mods: [Claude Code plugins overview](https://code.claude.com/docs/en/plugins/overview.md) and [Claude Code mods overview](https://code.claude.com/docs/en/plugins/mods/overview.md), verified 2026-11-08.
- Codex skills: [Codex skills documentation](https://learn.chatgpt.com/docs/build-skills.md), verified 2026-11-08.
- The shared library, chained launch path, core plugin and mods are from the author's own harness, described conceptually. The example skills and plugin are illustrations written for this page. The loading-layers diagram was drawn for this page; the sequence diagrams were made with archify.

{% include page_nav.html %}
