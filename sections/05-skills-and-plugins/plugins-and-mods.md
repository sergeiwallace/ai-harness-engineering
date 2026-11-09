---
title: "Plugins, marketplaces and mods"
parent: "Skills, commands and plugins"
nav_order: 3
---

{% include page_nav.html %}

# Plugins, marketplaces and mods

As of 2026-11-09.

A plugin is how a set of harness parts travels: skills, sub-agent profiles, hooks and MCP servers that only work well together, installed as one unit. This page covers the plugin layout and manifest, marketplaces, what a plugin costs in context, mods, and a pattern for sharing code across the skills in a plugin. It is part of [Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## What a plugin is

The docs define a plugin as "a directory of skills, agents, hooks, MCP servers, or other components that Claude Code installs and loads as one unit", and say to use one "when you want several skills, subagents, hooks, or MCP servers packaged as one unit".

### When to make one

| Situation | Plugin? | Why |
|---|---|---|
| One skill for one repository | No | A project skill in `.claude/skills/` is enough |
| A skill plus the hook that guards it | Yes | They must arrive together |
| The same parts in many repositories | Yes | One install, one version, everywhere |
| Parts for a whole team | Yes, in a marketplace | New people install once |

## The layout

```text
review-kit/
├── .claude-plugin/
│   └── plugin.json            # manifest
├── skills/
│   └── review/SKILL.md
├── agents/
│   └── security-reviewer.md   # sub-agent profile
├── hooks/
│   └── hooks.json             # hook registrations
└── .mcp.json                  # MCP servers the plugin starts
```

### The manifest

```json
{
  "name": "review-kit",
  "version": "1.4.0",
  "description": "Code review skill, a security reviewer sub-agent and a lint-before-commit hook.",
  "author": { "name": "Platform team" }
}
```

Raise the version on every change you ship, so people can tell which copy they have.

### How the parts load

![Sequence diagram with four lanes: New team member, Client, Marketplace repo and Context window. The person adds the marketplace example-org/platform-tools; the client reads marketplace.json and finds two plugins. The person installs review-kit; the client fetches version 1.4.0 with one skill, one agent profile and one hook, and says to run /reload-plugins. From then on the client puts the review skill and reviewer descriptions into the context window on every turn, and the lint hook runs before each commit as code.]({{ '/assets/images/05-skills-and-plugins/plugin-install-sequence.png' | relative_url }})
*Installing a plugin from a team marketplace, and what it costs on every turn afterwards.*

## Marketplaces

A marketplace is a catalogue of plugins that people add once and install from. A git repository with a marketplace file listing the plugins is enough, so a team can host its own.

### Worked example: a team marketplace

#### The catalogue

```json
{
  "name": "platform-tools",
  "owner": { "name": "Platform team" },
  "plugins": [
    { "name": "review-kit", "source": "./plugins/review-kit" },
    { "name": "release-kit", "source": "./plugins/release-kit" }
  ]
}
```

#### A new team member's session

```text
> /plugin marketplace add example-org/platform-tools
  Added marketplace platform-tools (2 plugins).

> /plugin install review-kit@platform-tools
  Installed review-kit 1.4.0: 1 skill, 1 agent, 1 hook.

> /reload-plugins
  Reloaded 1 plugin.
```

Three commands, and the new person has the same review skill, reviewer profile and lint hook as the rest of the team.

### Managing plugins

| Command | Does |
|---|---|
| `/plugin` | Lists, installs, enables and disables plugins |
| `/reload-plugins` | Applies plugin changes without a restart |
| `/skill-doctor` | Shows what each skill costs in context and how often it is used |

The same plugin format installs on claude.ai and in Cowork, where a different set of components loads. A cloud session does not load local plugins.

## The context cost

Every enabled plugin costs context on every turn, because the name and description of each model-invocable skill and agent in it stay loaded. Hooks and MCP servers add their own costs: a hook that injects text adds it on every event it fires on, and each MCP tool adds its name and schema.

### Worked example: an audit

| Plugin | Skills | Agents | MCP tools | Used last month | Action |
|---|---|---|---|---|---|
| review-kit | 1 | 1 | 0 | Daily | Keep |
| release-kit | 3 | 0 | 0 | Weekly | Keep |
| docs-helper | 6 | 2 | 12 | Never | Disable |
| old-linter | 2 | 0 | 0 | Never | Uninstall |

Disabling the two unused plugins removes eight skill descriptions, two agent descriptions and twelve tool schemas from every turn.

> Install what you use. Disable a plugin you use rarely and enable it for the session that needs it.

## Mods

A mod is a plugin with a hooks module: hooks written as JavaScript or TypeScript functions, which can change how Claude Code looks and behaves, for example by drawing panes in the terminal beside the chat. Mods need Claude Code 2.1.287 or later.

### Hooks module and shell hooks

| | Shell hook | Hooks module (mod) |
|---|---|---|
| Written in | Any language the shell can run | JavaScript or TypeScript |
| Runs as | A new process per event | Functions loaded into the client |
| Can draw in the terminal | No | Yes, panes and status areas |
| Typical use | Guards and checks | Interface changes and live panes |

#### Example: a context pane

A mod draws a small pane that shows how full the context window is and updates after each turn. The person sees the number without asking, and decides when to compact. Section 11 covers [the context window]({{ '/sections/11-context-window/' | relative_url }}).

## Shared code across skills

### The problem

Skills often include small command-line tools. Written one per skill, each parses options its own way, loads settings its own way and has its own test helpers. A bug fixed in one stays in the others.

### Plugin modules

One pattern that works is a plugin whose skills, MCP servers and hooks all import one shared library, so the code is written once. I call this a plugin module. That is my own name; Claude Code has no such term. The nearest vendor term is the hooks module above.

![Sequence diagram with five lanes: Developer, Shared library, Skill A tool, Skill B tool and MCP server. Two skill tools call settings.load and both get a wrong path on a second profile. The developer fixes the profile lookup in the library once and adds a test, which passes. After the next install, Skill A, Skill B and the MCP server each call settings.load and get the correct path, with no change to any of them.]({{ '/assets/images/05-skills-and-plugins/shared-library-fix-sequence.png' | relative_url }})
*A settings bug fixed once in the shared library. Every tool, hook and server that imports it gets the fix on the next install.*

#### Worked example: a skill tool on the shared library

```python
# A skill's tool: only its own logic, everything else from the library
from toolkit import entry, options, settings

@entry.command(help="List open issues older than N days.")
@options.flag("--days", type=int, default=14)
def main(days):
    cfg = settings.load()                 # same settings file as every tool
    for issue in cfg.tracker.open_issues(older_than=days):
        print(issue.id, issue.age_days, issue.title)
```

The tool has seven lines of its own. Option parsing, settings, error output and exit codes come from the library, so every tool behaves the same, and a fix in the library reaches all of them on the next install.

<details markdown="1">
<summary>A core plugin for two clients</summary>

In my harness one core plugin serves the same read-only tools to Claude Code and to Codex through an MCP server, so both clients query the same knowledge store the same way. The server is built on the same shared library as the skill tools. A tool added to the server is available in both clients at once.

</details>

## Pitfalls

- A plugin that bundles everything costs context in every session that enables it.
- A plugin version that never changes leaves people unsure which copy they run.
- Copying a helper into each skill multiplies every bug in it.

## Sources

- Plugins, marketplaces, the hooks module and mods: <https://code.claude.com/docs/en/plugins/overview.md> and <https://code.claude.com/docs/en/plugins/mods/overview.md>, verified 2026-11-08.
- `/plugin`, `/reload-plugins`, `/skill-doctor`: <https://code.claude.com/docs/en/commands.md>, verified 2026-11-09.
- The shared library, core plugin and context pane are from the author's own harness, described conceptually. The example plugin, marketplace, audit and tool code are illustrations written for this page.

{% include page_nav.html %}
