---
title: "Writing refusal messages"
parent: "The improvement loop"
grand_parent: "Building your own harness"
nav_order: 2
---

{% include page_nav.html %}

# Writing refusal messages

As of 2026-11-09. Sub-topic of [The improvement loop]({{ '/sections/17-building-a-harness/reflexivity-loop.html' | relative_url }}), in [Building your own harness]({{ '/sections/17-building-a-harness/' | relative_url }}).

When a hook, a script or a gate refuses an action, the agent reads the refusal message and decides what to do next. The message is the only part of the guard the agent sees. This page covers what a refusal message should contain, with before-and-after examples.

![Sequence diagram of two refusals of the same force push. With a bare refusal, the hook returns only "Blocked", the agent retries the same action with a different spelling, is blocked again, and hands the task back to you. With a refusal that names the next step, the hook explains why and says to push a new branch and open a pull request; the agent pushes a new branch and opens a pull request for review.]({{ '/assets/images/17-building-a-harness/refusal-messages-sequence.svg' | relative_url }}){: style="width:100%"}

*Figure 1. The same guard with two different messages. Diagram made with archify.*

## What a refusal message contains

1. What was refused.
   - Name the command, file or action, so the agent knows which step failed.
2. Why.
   - One sentence with the rule's reason. An agent that knows the reason is less likely to try the same thing in another form.
3. The allowed route.
   - The action that reaches the same goal within the rules.
4. Where to read more, if the rule is long.
   - A pointer to the context file section or the skill that covers it.

## Where refusals fit in the layers

A refusal message is guidance (layer 1) delivered by an enforcing layer (3 or 4), at the moment the rule applies.

| | A rule in a context file | The same rule in a refusal message |
|---|---|---|
| When the agent reads it | At the start of the session | At the moment it tries the action |
| Competes with | Everything else loaded since | Nothing; it is the latest message |
| If the agent forgets it | The action goes ahead | The action is still refused |

So the refusal message is the most reliable place to explain a rule. The context file can then shrink to a line that says the guard exists.

## Before and after

### A force push

```text
Before:
Blocked.

After:
Force push refused: `git push --force` rewrites a shared branch's history.
Push your work to a new branch instead (`git push -u origin <new-branch>`) and open a pull request.
```

### A generated file

```text
Before:
Error: protected path.

After:
Edit refused: generated/api_client.py is produced by `make gen` from schema/api.yaml.
Change schema/api.yaml, then run `make gen` to regenerate this file.
```

### Tests not run since the last edit

```text
Before:
Cannot finish.

After:
Finish refused: src/parse.py changed at 14:02 and the last passing test run was at 13:40.
Run `make test`, fix any failure, then finish.
```

## Common problems

| Problem | What the agent does | Fix |
|---|---|---|
| No reason given | Tries the same action with different syntax | Add one sentence with the reason |
| No allowed route | Stops and hands the task back to you | Name the action that reaches the goal |
| A wall of text | Misses the key line | Two or three lines; link to the long version |
| Blames the agent ("You broke the rules") | Apologises and repeats the attempt | State the fact and the next step |
| Message on standard output when the client reads standard error | Never sees the message | Check which stream your client passes back |

<details markdown="1">
<summary>Which stream the message goes to</summary>

In Claude Code, a hook that exits with code 2 has its standard error passed back to the agent. A message printed to standard output in that case does not reach the agent. Other clients differ, so check the documentation for the client you use.

</details>

## Testing a refusal message

A refusal message can be tested like any other output.

1. Write a test that feeds the guard a bad input.
   - Assert the exit code.
   - Assert that the message names the allowed route.
2. Run a real session that hits the guard.
   - Read the transcript and check that the agent's next action was the allowed route.

```bash
# Illustrative test of the force-push guard's message
out=$(echo '{"tool_input":{"command":"git push --force"}}' | ./tools/guard-push.sh 2>&1 >/dev/null)
rc=$?
[[ $rc -eq 2 ]] || { echo "expected exit 2, got $rc"; exit 1; }
[[ "$out" == *"new branch"* ]] || { echo "message does not name the allowed route"; exit 1; }
```

## Examples from my own harness

- Refusals in my harness name the rule, the reason and the allowed route.
- Several were rewritten after a transcript showed an agent retrying the refused action in another form.
  - The rewrite added the reason and the allowed route, following the format on this page.

To start: take the refusal message of one guard you have, and add the reason and the allowed route.

## Sources

- Claude Code hooks and exit code 2: <https://code.claude.com/docs/en/hooks.md>, not re-checked for this page.
- From the author's own harness, described conceptually.

{% include page_nav.html %}
