---
title: "Human gates, decisions and alignment"
permalink: /sections/14-human-gates/
nav_order: 24
has_children: true
has_toc: false
---

{% include page_nav.html %}

# Human gates, decisions and alignment

As of 2026-11-09.

Stop 10 on the talk path, about 3 minutes.

An agent that runs for hours needs to know which calls are its own and which belong to you. Decide that in advance: outward, destructive, spending and architectural actions wait for a person, and the agent decides everything else and records why. Ask rarely and well, because each question stops the work. Keep your original words, because a request that passes through several agents loses pieces on the way. Enforce the gates in the system, so they hold when the agent's attention is elsewhere.

![Sequence diagram of one task. The person asks the agent to build an export. The agent asks the authority test whether to add a new package; the test says it is architectural, so the agent writes options and a pick to the review queue. The agent asks whether to rename a helper; the test says decide and record. The person reads the queue in one sitting and rules no package; the ruling returns to the agent verbatim, and the agent returns the result with its decision log.]({{ '/assets/images/14-human-gates/who-decides-sequence.svg' | relative_url }}){: style="width:100%"}

*One task, two decisions. The agent checks each call against the authority test; only the architectural one reaches the person, through the review queue.*

## At a glance

| Idea | In one line | Go deeper |
|---|---|---|
| Human gate | a point where the agent stops for a person's decision | [What a human gate is](#what-a-human-gate-is) |
| Authority test | four questions that decide who decides | [Decision authority and escalation]({{ '/sections/14-human-gates/decision-authority.html' | relative_url }}) |
| Held decision | a question written as options and a recommendation, kept in one queue | [Held decisions and the review queue]({{ '/sections/14-human-gates/held-decisions.html' | relative_url }}) |
| Drift | requirements lost as work passes between agents | [Alignment with the original request]({{ '/sections/14-human-gates/alignment-and-drift.html' | relative_url }}) |
| Enforced gate | a rule, hook or check that holds whatever the model decides | [Permission rules and auto mode]({{ '/sections/14-human-gates/permission-rules.html' | relative_url }}) |
| Gate hook | your own code that refuses a tool call before it runs | [Writing a gate hook]({{ '/sections/14-human-gates/gate-hooks.html' | relative_url }}) |

> If an action is reversible and inside the agent's task, the agent decides and writes down why. Otherwise it asks once, with options.

### Pages in this section

1. This overview: the whole topic in one page.
2. [Decision authority and escalation]({{ '/sections/14-human-gates/decision-authority.html' | relative_url }}): the authority test, investigating before asking, and where to escalate.
   - [Held decisions and the review queue]({{ '/sections/14-human-gates/held-decisions.html' | relative_url }}): the template and the queue.
3. [Alignment with the original request]({{ '/sections/14-human-gates/alignment-and-drift.html' | relative_url }}): how drift happens and the check that catches it.
4. [Permission rules and auto mode]({{ '/sections/14-human-gates/permission-rules.html' | relative_url }}): allow, ask and deny rules, permission modes, and where they sit in the enforcement layers.
   - [Writing a gate hook]({{ '/sections/14-human-gates/gate-hooks.html' | relative_url }}): a PreToolUse hook from first draft to tested.

## What a human gate is

A human gate is a point where an agent must stop and get a person's decision before it continues. Everything else the agent does on its own.

- The gate names a kind of action, such as "any push to main" or "any spend over $20".
- It says who decides: the person, or for some design calls a stronger model.
- It says how the agent asks: a held decision in the review queue, or a prompt at the moment of action.

Where you put the gates decides two failure modes:

| Mistake | What happens |
|---|---|
| A missing gate | the agent publishes, deletes or spends without anyone agreeing |
| An extra gate | the agent waits hours on a call it could have made, and the person learns to approve without reading |

## Which actions need a person

Decide in advance which actions need a person, and write the list down. The usual list has five kinds:

| Kind | Examples | Why a person decides |
|---|---|---|
| Outward | publishing a page, sending a message, commenting on someone else's pull request | others see it, and it cannot be unsent |
| Destructive | deleting files, branches, data or cloud resources | it may not be reversible |
| Spending | a paid model batch, renting compute, a purchase | money is gone once spent |
| Architectural | a new design, a new dependency, a change to a public interface | it constrains every later change |
| Marked by the person | anything the person flagged as pending their call | they said so |

Contained technical calls go to the agent:

- which test to write first;
- how to name a helper;
- which of two equivalent libraries to use inside one module.

The agent records each such decision and its reason, so the person can audit them later without having been asked.

![Decision path. An agent wants to act. If the action is outward, destructive, spending or architectural, the person decides through a held decision. Otherwise the agent investigates first, then decides and records why, unless a gate still applies, in which case it goes to the person.]({{ '/assets/images/14-human-gates/gate-flow.svg' | relative_url }}){: style="width:100%"}

*The decision path for any single action.*

### Example: sorting five actions

An agent finishing a feature has five actions ahead of it:

| Action | Kind | Who decides |
|---|---|---|
| Rename a local variable | contained | the agent |
| Choose between two test layouts | contained | the agent, with a note on why |
| Add a new third-party package | architectural | the person |
| Delete a branch it created and has proved merged | destructive in form, but nothing is lost: the commits are on the remote | the agent, once a program has checked the proof |
| Post the release note to the team channel | outward | the person |

<details markdown="1">
<summary>Aside: why "reversible" is the test underneath the list</summary>

The five kinds on the list share one property: undoing them is hard, impossible, or visible to other people. That is why a deletion of a proved-merged branch can be exempt while a deletion of a database table cannot. When you meet an action the list does not name, ask whether a revert would fully undo it and whether anyone outside the task would see it. The [authority test]({{ '/sections/14-human-gates/decision-authority.html#question-2-is-it-outward-destructive-spending-or-architectural' | relative_url }}) spells this out.

</details>

## Asking too often is also a failure

Each question costs the person time and stops the work until they answer. An agent that asks about every choice trains the person to approve without reading. Two habits keep questions rare:

1. Investigate before asking.
   - Read the error in full, check the docs and the code, and look for an existing script or automated route.
   - Try the reversible fix.
   - Ask only if a gate still applies afterwards.
2. Apply the authority test.
   - Claimed by the person, outward, destructive, spending, architectural, or dropping a requirement: ask.
   - Otherwise: decide, and record the decision and its reason.

[Decision authority and escalation]({{ '/sections/14-human-gates/decision-authority.html' | relative_url }}) gives the full test.

| Signal | What it means |
|---|---|
| The person answers most questions with "yes, go ahead" | the agent is asking about calls it owns |
| Questions arrive one at a time during the day | they belong in a review queue |
| A question has no options | the agent has not investigated yet |

### Example: a blocked agent that did not need to ask

A test run fails because a login token expired.

1. The tempting move: stop and ask the person to log in.
2. The investigating move: check whether the harness has an automated login for that service, run it, and confirm the token works by calling the service.
3. Only if no automated route exists does the agent ask, and then it names the one step that needs a person.

```text
agent> Integration tests fail: 401 Unauthorized from the storage API.
agent> Checking for an automated login route in the harness docs... found one.
agent> Ran the login. Verifying by calling the service: list-buckets returned 200.
agent> Re-ran the tests: 48 passed. No question for the person.
```

## Asking well: held decisions

When an agent does need a person, it writes the question as a held decision that can be answered in one read:

- Context: the task it serves, what the agent investigated, the evidence.
- Options: two or more, each with pros and cons.
- Recommendation: the reasoning first, then the pick.
- Consequences: what happens next under each option, with cost and time.

The question goes into one review queue, a file or folder every agent writes to, so the person answers several at once when they have time. A file survives the end of the session; a chat message does not. [Held decisions and the review queue]({{ '/sections/14-human-gates/held-decisions.html' | relative_url }}) has the template.

### Example: a question that is easy to answer

| Poor question | Answerable question |
|---|---|
| "Should I use a queue here?" | "The import job times out on large files. Option A: split the file and run parts in parallel (small change, no new service). Option B: add a job queue (handles every future large job, adds a service to run). I recommend A, because the only large files today are monthly exports. Choosing B adds about a day of setup." |

![Sequence diagram. A blocked agent writes context and evidence, options with pros and cons, and a recommendation into a decision file, and adds a row with its path to the review queue. The person reads the queue, newest first, and writes a ruling in their own words into the file. The agent reads the ruling verbatim, marks the item resolved and reports that it acted on it.]({{ '/assets/images/14-human-gates/held-decision-sequence.svg' | relative_url }}){: style="width:100%"}

*A held decision, from question to ruling.*

## Alignment: keep the person's words

Agents drift from the original request as work passes through several hands:

1. The person asks for three things.
2. The lead agent's brief to a sub-agent summarises them as two.
3. The sub-agent's report says "done".
4. Each step looked reasonable and every check was green, but one requirement is gone.

The fix keeps the original words and checks each result against them:

1. Store the request verbatim, with a hash.
2. Carry it unchanged into every brief.
   - The task framing for each agent goes in a separate block marked non-authoritative.
3. Before calling work complete, run a drift check.
   - Every requirement the person named is either delivered or explicitly escalated.

[Alignment with the original request]({{ '/sections/14-human-gates/alignment-and-drift.html' | relative_url }}) walks through the mechanism.

![Sequence diagram. The person asks the lead agent for A, B and C. The lead briefs a sub-agent with A and B plus the verbatim request. The drift check fails because C is missing, the lead writes a held decision, the person says build C now, and a second pass ends with the drift check passing.]({{ '/assets/images/14-human-gates/drift-sequence.svg' | relative_url }}){: style="width:100%"}

*A dropped requirement, caught by the drift check and restored in a second pass.*

### Example: the dropped feature

1. A person asks for "a settings page with a dark mode toggle and an export button". The plan lists both.
2. An agent finds the export button needs a backend endpoint that does not exist yet.
   - It defers the button "for a follow-up" and ships the page.
3. The tests pass and the review is clean, because both check what was built.
4. The drift check compares the result with the original request:

```text
requirement          evidence                     status
dark mode toggle     toggle test passes           delivered
export button        agent decision: deferred     FAIL: dropping a requirement is the person's call
```

5. The agent writes a held decision with two options: build the endpoint now (half a day), or ship without the button and file a dated task. The person picks the first.

## Enforcing gates: layers, permissions and auto mode

A gate held only by an instruction is followed less reliably as a session grows long, the agent juggles several tasks, or the context window fills. A systems approach puts each gate into the system, in layers:

1. Instructions in context files, skills and prompts.
   - "Ask before you push." This guides the model and is the second line of defence.
2. Workflows and tools that build the step in.
   - A merge skill that opens a pull request and never pushes to main.
3. Hooks, scripts and linters that refuse or flag at the moment of action.
   - A PreToolUse hook or a deny rule that refuses a force-push.
4. Validation and tests before a change lands.
   - Branch protection or a pre-merge gate that rejects an unreviewed change.

Layers 2 to 4 do the enforcing. [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) covers layer 3 in general.

### What Claude Code enforces itself

In Claude Code, a tool call meets three parts:

1. A PreToolUse hook, your own code, runs before the permission prompt.
   - It can refuse the call, force a prompt, or say nothing.
2. The permission rules from `/permissions` and the `permissions` block in `settings.json`.
   - Deny is checked first, then ask, then allow; the first match decides.
   - A matching deny or ask rule applies even if the hook allowed the call.
3. The permission mode, for calls no rule matched.
   - `default` prompts on first use of each tool; `acceptEdits` accepts file edits; `plan` explores without editing.
   - `auto` replaces most prompts with a background classifier that checks each action against your request.

Codex has the same two ideas as an approval policy and a sandbox mode (not yet verified against the Codex docs for this page). [Permission rules and auto mode]({{ '/sections/14-human-gates/permission-rules.html' | relative_url }}) has the details and worked configurations, and [Writing a gate hook]({{ '/sections/14-human-gates/gate-hooks.html' | relative_url }}) builds a hook step by step.

![Sequence diagram with two alternatives. A push to main reaches the PreToolUse hook, which denies it and tells the agent to open a pull request. A push to a branch passes the hook with no decision, matches an ask rule, the person is prompted and says yes once, and the push runs.]({{ '/assets/images/14-human-gates/permission-layers-sequence.svg' | relative_url }}){: style="width:100%"}

*One push, through the permission layers: the hook refuses a push to main, and an ask rule prompts for a push to a branch.*

<!-- IMAGE NEEDED: a screenshot of a Claude Code permission prompt in the terminal, showing a Bash command and the allow/deny choices; search: "Claude Code permission prompt", "Claude Code allow command dialog" -->

### Example: one gate, moved into the system

1. A team writes "never force-push to main" in its `CLAUDE.md`.
2. An agent in a long session, rebasing to fix a conflict, force-pushes anyway.
3. The team fixes the system rather than adding another line to the context file:
   - a deny rule, `"Bash(git push --force *)"`, in `.claude/settings.json`;
   - a hook that refuses any push whose target is main, with a refusal naming the pull request route;
   - branch protection on main, so even a push that slips through is rejected.
4. The next time, the call is refused before it runs, and the agent reads the refusal and opens a pull request.

{: .warning }
> A Bash deny rule matches the command text. `Bash(git push --force *)` does not match `git push -f origin main`, and a deny rule does not match the same program inside `sh -c`. That is why the hook, which reads the whole command, and branch protection, which checks on the server, sit behind it.

### Example: a starter rule set

```json
{
  "permissions": {
    "allow": ["Bash(npm run test *)", "Bash(git commit *)"],
    "ask":   ["Bash(git push *)"],
    "deny":  ["Read(./.env)", "Read(./secrets/**)"]
  }
}
```

| Rule | Effect |
|---|---|
| `allow` tests and commits | they run without a prompt |
| `ask` on pushes | every push prompts, even in a mode that would otherwise run it |
| `deny` on the secrets file and folder | Claude's file tools cannot read them |

Commit this as `.claude/settings.json` so every session in the repository gets the same gates.

## Examples from my own harness

- A decision procedure with an authority test that decides who decides. Every decision and blocker passes through it before it may reach me; most end as a recorded agent decision.
- My original request is stored verbatim with a hash, carried into every brief, and checked by a gate that fails any agent decision that drops, narrows or contradicts something I asked for.
- Held decision documents collect every question for me in one review queue, each written as options with pros and cons and a recommendation, so I answer them in one sitting.
- Outward actions such as external comments stay gated per action, while pushes and merges in my own repositories are ungated once the local test gate is green.

To start: write the list of actions an agent must never take alone, then enforce the one you fear most with a deny rule or a hook, and keep the list in your context file as the second layer.

## Pitfalls and lessons

- Gating too much leaves agents waiting on calls they could make. Push contained decisions to the agents and keep a record of each one.
- A gate written only as an instruction is followed less often as a session grows long. Move the gates you care about into permission rules, hooks or checks.
- A broad allow rule added to stop prompts, such as `Bash(*)`, removes every gate the prompts were providing.
- A recommendation without the options behind it is hard to overrule. Ask for the options even when the agent is confident.
- The drift check came from applying the systems approach to one failure: an agent's decision dropped a feature the request had named, with every other gate green. The fix went into the system as a gate, with a context-file line as the second layer.

## Sources

- Permission rules, their order, the permission modes and how hooks interact with them: <https://code.claude.com/docs/en/permissions.md>, verified 2026-11-09.
- Codex approval policy and sandbox mode: not yet verified against the primary source for this page.
- The decision procedure, the stored request and held decisions: from the author's own harness, described conceptually.

{% include page_nav.html %}
