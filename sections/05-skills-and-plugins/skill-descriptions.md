---
title: "Writing skill descriptions"
parent: "Skill anatomy and loading"
grand_parent: "Skills, commands and plugins"
nav_order: 1
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Writing skill descriptions

The description is the only part of a skill that is always in the context window, and it is what the model reads when it decides which skill to use. A skill with a vague description is not picked, or is picked for the wrong request. This page is a sub-topic of [Skill anatomy and loading]({{ '/sections/05-skills-and-plugins/skill-anatomy.html' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## How the model picks a skill

![Sequence diagram with four lanes: Person, Model, review and audit. Before: the person asks the model to check the change before a push; the review description, Review code, has no scope or trigger; the audit description, Audit code for problems, is the closer match for the word check, so the model runs audit, which reads the whole module for several minutes. After, in a new session: the model matches the request to the rewritten review description, the current diff before a commit or pull request, and gets three findings in a minute; audit is skipped because its description says to use it only when the user asks for an audit.]({{ '/assets/images/05-skills-and-plugins/skill-pick-sequence.png' | relative_url }})
*The same request before and after the descriptions were rewritten. The model sees only the descriptions when it chooses.*

When it chooses, the model has the conversation so far and the name and description of every model-invocable skill.

- It does not see any skill body until it has chosen.
- Anything it needs to tell two skills apart must be in their descriptions.
- A person-only skill is not in the list at all, so it is never chosen this way.

## The three parts of a good description

### What it does

One sentence, with a verb and an object.

- Name the output, so the model can tell whether the request wants it.
- Name the scope: the current diff, one module, the whole repository.
- Example: "Write a pull request description for the current branch."

### When to use it

The model matches the request against these words, so use the ones a person would type.

- Trigger words: "PR description", "write up the PR".
- Workflow moments: "before opening a pull request".
- Example: "Use when the user asks for a PR description, or before opening a pull request."

### When to use something else

Needed only when a neighbouring skill is close.

- Name the neighbour and the case that belongs to it: "For a full module review, use audit."
- Add the pointer to both skills, so each says where the other's requests go.

#### Template

```text
<Verb> <object> <scope>. Use when <trigger words or workflow moment>.
[For <neighbouring case>, use <other skill>.]
```

## Before and after

| Skill | Before | After | What changed |
|---|---|---|---|
| `review` | Review code | Review the current diff before a commit or pull request. For a whole module, use audit | Scope and a pointer to the neighbour |
| `audit` | Audit code for problems | Audit a whole module for design problems. Use only when the user asks for an audit | Scope and an explicit trigger |
| `pr-description` | PR helper | Write a pull request description for the current branch. Use before opening a pull request | Output named, moment named |
| `deploy-staging` | Deploys things | Deploy the current branch to the staging environment. Use when the user says deploy to staging | Target named, trigger words |

### Worked example: diagnosing a wrong pick

| Stage | What happened |
|---|---|
| Symptom | A person types "can you check this before I push" and the agent runs `audit`, which reads the whole module and takes several minutes |
| Cause | Both descriptions say "code" and neither says when; "check" is as close to "audit" as to "review" |
| Fix | Both rewritten with scope and triggers, as in the table above |
| Check | The same request in a new session now runs `review` |

> Test a description change in a new session. In the session where you edited it, the old description may still be in the window.

## Length

A description is paid on every turn for every enabled model-invocable skill, so one or two sentences is the usual size. Words in the window on every turn:

| Words per description | 10 skills | 30 skills | 100 skills |
|---|---|---|---|
| 25 | 250 | 750 | 2,500 |
| 100 | 1,000 | 3,000 | 10,000 |

<details markdown="1">
<summary>Where the detail goes instead</summary>

A description that tries to list every case grows long and still misses some. Put the cases in the body, where they cost only on use. The description needs only enough to choose correctly; the body handles what to do once chosen.

</details>

## Descriptions for person-only skills

A skill with `disable-model-invocation: true` is never picked by the model, so its description is read by people scanning the command menu.

| Write | Example |
|---|---|
| What it does | Push a release tag and publish the changelog |
| What it touches | The remote repository and the public release page |
| Whether it can be undone | Tags can be deleted; a published release has been seen |

## Testing a description

1. Start a new session, so the window holds the new text.
2. Ask for the task in two or three ways a person would, without naming the skill.
3. Confirm the model starts the skill you expect each time.
4. Ask for the neighbouring skill's task and confirm it goes there.
5. Check length and format in a test: the [frontmatter test on the overview]({{ '/sections/05-skills-and-plugins/' | relative_url }}#a-frontmatter-test) fails a description outside 20 to 300 characters.

Steps 1 to 4 check that the model picks the right skill; step 5 checks the text itself on every commit.

## Checklist

1. Does the first sentence name the output?
2. Does it say when, in the words a person would use?
3. If a neighbour is close, does it point to the neighbour?
4. Is it under about 50 words?
5. Did you test it with a real request in a new session?

## Pitfalls

- Two skills whose descriptions share their main words, such as "code", "check" or "docs".
- A description that lists every case, which costs words on every turn and still misses some.
- Testing a change in the session where it was edited.
- A model-only skill (`user-invocable: false`) with a vague description: the description is its only way to be started.

## Sources

- Skill descriptions and how the model chooses a skill: <https://code.claude.com/docs/en/skills.md>, verified 2026-11-09.
- The example descriptions and the wrong-pick case are illustrations written for this page.

{% include page_nav.html %}
