---
title: "Skills, commands and plugins"
nav_order: 15
has_children: true
has_toc: false
---

{% include page_nav.html %}

# Skills, commands and plugins

As of 2026-11-09.

Stop 4 on the talk path, about 1.5 minutes. [Skills and plugins in detail]({{ '/sections/05-skills-and-plugins/detail.html' | relative_url }}) covers the file format, invocation settings, chaining, plugins, mods and shared code.

## What a skill is

A skill is a folder with a `SKILL.md` file of instructions and, optionally, scripts and reference files. Its short description is always in the context, so the model knows the skill exists. Its body loads only when the skill is used. That makes a skill the right home for a procedure that is long, needed sometimes, and should run the same way each time: your release checklist, your way of writing a bug report, your test procedure.

In Claude Code, custom slash commands have been merged into skills: a skill named `release` is started with `/release`. Skills follow the Agent Skills open standard, which Codex also uses.

### Example: a small skill

```markdown
---
name: bug-report
description: Write a bug report from a failing test or an error the user pastes.
---
1. Reproduce the failure and record the exact command and output.
2. Find the smallest input that still fails.
3. Write the report with: summary, steps, expected, actual, environment.
4. Save it to docs/bugs/<date>-<slug>.md and show the path.
```

You type `/bug-report` with the error, or the model picks the skill itself when you say "file this as a bug". Either way it runs the same four steps.

## Who can start a skill

By default both you and the model can start a skill. Two frontmatter settings change that:

- `disable-model-invocation: true`: only you can start it. Use this for skills with side effects you must choose, such as publishing, sending or merging.
- `user-invocable: false`: only the model can start it. Use this for background procedures the model should pick up when they fit.

Codex has a similar setting, `allow_implicit_invocation: false`.

### Example: choosing the setting

A skill that runs your test procedure should stay model-invocable, so the agent uses it whenever it changes code. A skill that publishes a release should be person-only, so the agent never decides on its own to ship.

## Chaining skills

You can start one message with several skills, up to six, and Claude Code loads them all. A skill's instructions can also tell the agent to run another skill when it reaches a step, which builds a pipeline out of small parts. See [deep dive: chaining]({{ '/sections/05-skills-and-plugins/detail.html' | relative_url }}#chaining-skills).

## Plugins

A plugin is "a directory of skills, agents, hooks, MCP servers, or other components that Claude Code installs and loads as one unit", usually from a marketplace. Use one when you want to share a set of parts that work together. A mod is a plugin that changes how Claude Code looks and behaves.

Every enabled plugin costs context on every turn, because the name and description of each model-invocable skill and agent in it stay loaded. Install what you use, and disable what you do not.

### Example: a review plugin

A team packages its review skill, a sub-agent profile for a security reviewer, and a hook that runs the linter before each commit, as one plugin. A new team member installs it with one command and gets all three.

## Pages for this topic

- [Skills and plugins in detail]({{ '/sections/05-skills-and-plugins/detail.html' | relative_url }}): the skill format, how loading works, invocation settings, writing good descriptions, chaining, plugins and marketplaces, mods, shared code across skills, and managing a growing set.

{% include page_nav.html %}
