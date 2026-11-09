---
title: "Invocation and chaining"
parent: "Skills, commands and plugins"
nav_order: 2
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Invocation and chaining

A skill can be started by a person, by the model, or by another skill. This page covers the settings that decide who may start a skill, how to choose them, the trap that breaks chains silently, and how to build a pipeline out of small skills. It is part of [Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Who can start a skill

### Three ways in

1. A person types `/name`, and the client loads the skill.
2. The model matches a request to a skill's description and loads it.
3. Another skill, a sub-agent's brief or a hook's message says "run `/name`".
   - The agent reads that as an instruction.
   - It is the model that starts the skill, so this is a form of way 2.
   - That detail decides how chains behave.


### The settings

| Frontmatter | Person | Model | In the menu | Description in the window |
|---|---|---|---|---|
| (default) | Yes | Yes | Yes | Yes |
| `disable-model-invocation: true` | Yes | No | Yes | No |
| `user-invocable: false` | No | Yes | No | Yes |

- A person-only skill also takes its description out of the window, so it costs nothing until a person starts it.
- A model-only skill stays out of the `/` menu, so its description is its only way to be started.
- Codex has a similar setting, `allow_implicit_invocation: false`.

```yaml
---
name: publish-release
description: Push a release tag and publish the changelog.
disable-model-invocation: true
---
```

### Choosing a setting

#### Decision table

| Question | If yes |
|---|---|
| Does it change something outside the checkout: push, publish, send, delete, pay? | Person-only, unless something else in the harness calls it |
| Should the model use it whenever it fits, like tests or a review checklist? | Default |
| Is it a background procedure people never type, like a formatting convention? | Model-only |
| Does another skill, a hook or a brief call it? | Must stay model-invocable |

#### Worked example: four skills

| Skill | Side effects | Called by others | Setting |
|---|---|---|---|
| `/run-tests` | None | Yes, by the review skill | Default |
| `/pull-latest` | Can overwrite local work | No | Person-only |
| `/publish-release` | Customers download the result | No | Person-only |
| `commit-style` | None | Read by the commit step | Model-only |

## The chain trap

A person-only skill cannot be started by the model, and everything except a person goes through the model. So a skill that calls a person-only skill cannot start it.

![Sequence diagram with four lanes: Person, Model, Client and /publish-release, which is person-only. The person types /release-day; the model runs steps 1 to 3 through the client. At step 4 the model asks to run /publish-release; the client answers that it is not available because it is person-only; the model reports release day complete, though step 4 never ran. The fix: the skill is made model-invocable with a confirm step. Now the model runs it, the skill shows the tag and changelog to the person and waits; the person says yes; the skill reports published.]({{ '/assets/images/05-skills-and-plugins/chain-trap-sequence.png' | relative_url }})
*The chain trap and its fix. The purple arrow is the model reporting the run as complete when step 4 did not run.*

What happens in the session:

1. The agent reaches step 4 and reads "run `/publish-release`".
2. It cannot start a person-only skill.
3. No error is raised, and the agent can report the run as complete without step 4.

### How to keep the gate

Keep the called skill model-invocable and put the gate where it holds:

1. A confirmation step inside the skill: "Show the tag and the changelog and wait for the person to say yes."
2. A hook on the dangerous command itself, such as the tag push, which refuses unless a condition holds. Section 6 covers [hooks]({{ '/sections/06-hooks-scripts-and-linters/hooks.html' | relative_url }}).

> The gate belongs on the dangerous action. A setting on the skill controls only one of the ways to reach that action.

### Worked example: a test that finds the trap

A test can find the trap before it reaches a session:

1. Read the frontmatter of every `SKILL.md` and collect the person-only skills.
2. Search every other skill body, brief template and hook message for `/<name>` of each one.
3. Fail the test with the file that makes the call.

The [frontmatter test on the overview]({{ '/sections/05-skills-and-plugins/' | relative_url }}#a-frontmatter-test) does this in a dozen lines.

## Chaining skills

### Several skills in one message

Claude Code lets a person start one message with several skills, up to six.

```text
> /review /pr-description get this ready to merge
```

- Both skills load before the task.
- The agent applies them in turn: the review first, then the description.
- The person chose the order, so nothing in the skills needs to call the other.

### A skill that calls a skill

A skill's step can tell the agent to run another skill.

- This is a convention; the docs do not describe it.
- It works because the agent reads the step as an instruction and starts the named skill.
- The called skill must be model-invocable, for the reason in [the chain trap](#the-chain-trap).

```markdown
---
name: ship
description: Take the current branch from finished code to an open pull request. Use when the user says ship it.
---
1. Run /run-tests. Stop if any test fails and report the failure.
2. Run /review on the diff. Fix every finding marked high.
3. Run /pr-description and open the pull request with its text.
4. Report the pull request URL.
```

![Sequence diagram of the /ship skill: the model runs /run-tests, then /review, fixes the high finding and re-runs the tests, then runs /pr-description and reports the open pull request.]({{ '/assets/images/05-skills-and-plugins/ship-chain-sequence.png' | relative_url }})
*The skill above during a run. Step 2 loops back to the test skill after a fix.*

### Pipeline patterns

| Pattern | Shape | Use for |
|---|---|---|
| Sequence | A then B then C | Fixed procedures like ship or release |
| Fan-out | A starts B once per item | One worker per file or per task |
| Gate | A runs B, stops on failure | A test or review before a merge |
| Loop | A runs B until a check passes | Fix and re-test until green |

#### Fan-out in practice

1. `/orchestrate` plans the work and splits it into tasks.
2. For each task it runs `/delegate`.
   - `/delegate` writes the brief from one template.
   - It starts the worker in its own checkout.
3. Each finished task goes through `/review`.
4. `/merge` lands the reviewed results one at a time.

This is the chain in my harness. Every launch goes through the one delegation skill, so a fix to how briefs are written applies to every launch. Section 10 covers [orchestration]({{ '/sections/10-orchestration/' | relative_url }}).

### How chains fail

| Failure | What you see | Fix |
|---|---|---|
| Called skill is person-only | Chain stops; work reported done | Keep it model-invocable; gate the action |
| A step's output is unclear | Next skill starts on the wrong input | Each step names its output file or value |
| Gate skipped under pressure | Merge without tests | Gate in a hook as well as in the skill |
| Long chain fills the window | Later steps forget earlier rules | Fork heavy steps; keep the chain short |

<details markdown="1">
<summary>Why each step should name its output</summary>

| Step written as | Next step |
|---|---|
| "Run /review" | Has to guess where the findings are |
| "Run /review and save findings to review.md" | Reads `review.md` |

When a chain breaks, the saved outputs also show which step it broke at.

</details>

## Pitfalls

- A person-only skill called by another part of the harness stops the chain without an error.
- A gate that lives only in a skill is skipped whenever the skill is.
- Chains of more than a few skills lose early instructions as the window fills.

## Sources

- Invocation settings and several skills in one message, up to six: [Claude Code skills documentation](https://code.claude.com/docs/en/skills.md) and [Claude Code commands reference](https://code.claude.com/docs/en/commands.md), verified 2026-11-09.
- Codex `allow_implicit_invocation`: [Codex skills documentation](https://learn.chatgpt.com/docs/build-skills.md), verified 2026-11-08.
- The chained launch path is from the author's own harness, described conceptually. The example skills are illustrations written for this page.

{% include page_nav.html %}
