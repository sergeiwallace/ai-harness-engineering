---
title: "Writing skill descriptions"
parent: "Skill anatomy and loading"
grand_parent: "Skills, commands and plugins"
nav_order: 1
---

{% include page_nav.html %}

# Writing skill descriptions

As of 2026-11-09.

The description is the only part of a skill that is always in the context window, and it is what the model reads when it decides which skill to use. A skill with a vague description is not picked, or is picked for the wrong request. This page is a sub-topic of [Skill anatomy and loading]({{ '/sections/05-skills-and-plugins/skill-anatomy.html' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## How the model picks a skill

![Sequence diagram with four lanes: Person, Model, review and audit. Before: the person asks the model to check the change before a push; the review description, Review code, has no scope or trigger; the audit description, Audit code for problems, is the closer match for the word check, so the model runs audit, which reads the whole module for several minutes. After, in a new session: the model matches the request to the rewritten review description, the current diff before a commit or pull request, and gets three findings in a minute; audit is skipped because its description says to use it only when the user asks for an audit.]({{ '/assets/images/05-skills-and-plugins/skill-pick-sequence.png' | relative_url }})
*The same request before and after the descriptions were rewritten. The model sees only the descriptions when it chooses.*

The model sees the request and the list of descriptions, nothing more. It does not see the body until it has chosen. Everything the model needs to choose correctly must therefore be in the description.

## The three parts of a good description

### What it does

One sentence, with a verb and an object: "Write a pull request description for the current branch." Name the output, so the model can tell whether the request wants that output.

### When to use it

The words a person would say, and the moments in a workflow where the skill applies: "Use when the user asks for a PR description, or before opening a pull request." Trigger words matter, because the model matches the request against them.

### When to use something else

Needed only when a neighbouring skill is close: "For a full module review, use audit." This one line removes most wrong picks between two similar skills.

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

#### The symptom

A person types "can you check this before I push" and the agent runs `audit`, which reads the whole module and takes several minutes.

#### The cause

Both descriptions say "code" and neither says when. The model has no basis to prefer `review`, and "check" is as close to "audit" as to "review".

#### The fix

Rewrite both with scope and triggers, as in the table above. Then test with the same request in a new session: the agent now picks `review`.

> Test a description change in a new session. In the session where you edited it, the old description may still be in the window.

## Length

A description is paid on every turn for every enabled skill. One or two sentences is enough for almost every skill. Thirty skills at 25 words is 750 words in every turn; thirty at 100 words is 3,000.

| Words per description | 10 skills | 30 skills | 100 skills |
|---|---|---|---|
| 25 | 250 | 750 | 2,500 |
| 100 | 1,000 | 3,000 | 10,000 |

<details markdown="1">
<summary>Where the detail goes instead</summary>

A description that tries to list every case grows long and still misses some. Put the cases in the body, where they cost only on use. The description needs only enough to choose correctly; the body handles what to do once chosen.

</details>

## Descriptions for person-only skills

A skill with `disable-model-invocation: true` is never picked by the model, so its description is read by people in the command menu. Write it for a person scanning a list: what it does and what it touches. "Push a release tag and publish the changelog" tells the person this one has side effects.

## Checklist

1. Does the first sentence name the output?
2. Does it say when, in the words a person would use?
3. If a neighbour is close, does it point to the neighbour?
4. Is it under about 50 words?
5. Did you test it with a real request in a new session?

## Sources

- Skill descriptions and how the model chooses a skill: <https://code.claude.com/docs/en/skills.md>, verified 2026-11-09.
- The example descriptions and the wrong-pick case are illustrations written for this page.

{% include page_nav.html %}
