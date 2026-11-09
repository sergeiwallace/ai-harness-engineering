---
title: "Testing at the real entry point"
parent: "Testing agent-written code"
nav_order: 1
redirect_from:
  - /sections/12-testing/detail.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Testing at the real entry point
{: .no_toc }

A behaviour-first test drives the code through the path a user or another program actually takes, and asserts the outcome the acceptance criteria name. This site uses it as the default test for agent-written code, because an agent cannot make it pass by mirroring its own implementation.

This page covers, in order:

1. what counts as an entry point;
2. why the classic test pyramid shifts when agents write the tests;
3. how to write the test, with a full worked example of a Claude Code hook;
4. mocks, and when to use them;
5. the value gate for function-level tests, including property-based tests;
6. how to order spec, tests and code;
7. pitfalls.

The next page, [Test tampering and oracle integrity]({{ '/sections/12-testing/oracle-integrity.html' | relative_url }}), covers what happens when an agent cannot make such a test pass.

<details markdown="1">
<summary>Contents of this page</summary>

* TOC
{:toc}

</details>

## What counts as an entry point

An entry point is the boundary where the outside world meets the code. The test calls it the same way the outside world does, with the same kind of input, and reads back the same kind of output.

| Kind of code | Real entry point | What the test asserts |
|:--|:--|:--|
| Command-line tool | The command with real arguments, in a temporary home directory | Exit code, printed output, files written |
| Web service | An HTTP request to the route | Status, response body, stored state |
| Hook or plugin | The host's JSON payload on standard input | The decision it returns and its exit code |
| Rendered page | A browser loading the page | Visible text, layout, the result of a click |
| Library with a public API | The public function a caller imports | The documented return value or error |

For a library, the public function is the entry point, so a behaviour-first test of a library calls that function.

## The classic testing pyramid

<figure>
  <img src="{{ '/assets/images/12-testing/classic-testing-pyramid.png' | relative_url }}" style="max-width: 381px; width: 100%;" alt="The classic testing pyramid: a wide base labelled Unit, a middle band labelled Integration, and a small top labelled E2E.">
  <figcaption>The classic testing pyramid: most tests at the unit level, few end to end. Image by Mike Wacker, public domain, via Wikimedia Commons: <a href="https://commons.wikimedia.org/wiki/File:Testing_Pyramid.png">File:Testing_Pyramid.png</a>, from the Google Testing Blog post <a href="https://testing.googleblog.com/2015/04/just-say-no-to-more-end-to-end-tests.html">"Just Say No to More End-to-End Tests"</a> (2015).</figcaption>
</figure>

The pyramid was drawn for human teams, where unit tests were cheap to write and end-to-end tests were slow and flaky. Two things change when an agent writes the tests:

- Writing a test now costs almost nothing, so the unit base grows without limit.
- The agent writes that base by reading its own implementation, so the tests mirror the code instead of checking the intent.

So the weight moves up:

| Layer | Classic pyramid | With agent-written code |
|:--|:--|:--|
| End-to-end | A few slow browser tests | Still few; a live run per feature where one exists |
| Entry point and integration | A middle band | The default: one test per acceptance criterion, through the real command or route |
| Unit | The wide base | Only tests that pass the [value gate](#the-value-gate) |

Kent C. Dodds's 2019 advice, "Write tests. Not too many. Mostly integration.", described the same shift before agents. The cost of end-to-end tests has not gone away (see the flakiness aside on the [overview]({{ '/sections/12-testing/#heavy-test-runs-on-a-shared-machine' | relative_url }})), so "drive the entry point" usually means a fast, hermetic run of the real command or route, not a full browser session for every case.

<!-- IMAGE NEEDED: the "testing trophy" shape (static, unit, integration largest, end to end) next to the classic pyramid; search: testing trophy Kent C Dodds, testing trophy vs pyramid -->

## Writing the test

### Step by step

1. Read the acceptance criterion and name the behaviour in one sentence a user would recognise.
2. Find the entry point that behaviour is reached through.
3. Build the smallest real environment it needs: a temporary directory, a test database, a local server.
4. Call the entry point with realistic input.
5. Assert the observable outcome, not the intermediate values.
6. Check that the test fails when the behaviour breaks.
   - Break the code on purpose (comment out the line that does the work, or return early), run the test, see it fail, then restore the code.
   - A test that passes on broken code checks nothing.

### Worked example: a hook that blocks a dangerous command

Claude Code can run a hook before every tool call. The hook receives a JSON description of the call on standard input and can block it. The contract, from Claude Code's hooks reference:

| Hook result | Effect on the tool call |
|:--|:--|
| Exit code 0 | The call proceeds |
| Exit code 2, reason on standard error | The call is blocked; Claude receives the reason as the tool's error |
| Any other non-zero exit code | A non-blocking error; the call still proceeds |
| Exit code 0 with JSON `permissionDecision: "deny"` on standard output | The call is blocked, with the JSON reason |

Suppose an agent writes a hook that blocks `git push --force`.

#### The hook

```bash
#!/usr/bin/env bash
# tools/block-force-push.sh: refuse force pushes before they run
cmd=$(jq -r '.tool_input.command // empty')
if [[ "$cmd" =~ git[[:space:]]+push.*(--force|-f([[:space:]]|$)) ]]; then
  echo "Blocked: force push is not allowed; push normally or ask the owner." >&2
  exit 2
fi
exit 0
```

#### Registering it

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "${CLAUDE_PROJECT_DIR}/tools/block-force-push.sh" }
        ]
      }
    ]
  }
}
```

The `matcher` limits the hook to the `Bash` tool, so it does not run on file edits.

#### The test an agent tends to write

```python
def test_is_force_push():
    assert is_force_push("git push --force origin main")
    assert not is_force_push("git push origin main")
```

It checks a helper. It passes even if the hook never calls the helper, reads the wrong JSON field, or exits with code 1, which does not block.

#### The behaviour-first test

```python
import json, subprocess

def run_hook(command):
    payload = {"hook_event_name": "PreToolUse", "tool_name": "Bash",
               "tool_input": {"command": command}}
    return subprocess.run(
        ["./tools/block-force-push.sh"],
        input=json.dumps(payload), capture_output=True, text=True,
    )

def test_force_push_is_blocked():
    result = run_hook("git push --force origin main")
    assert result.returncode == 2
    assert "force push" in result.stderr.lower()

def test_short_flag_is_blocked():
    assert run_hook("git push -f origin main").returncode == 2

def test_normal_push_is_allowed():
    assert run_hook("git push origin main").returncode == 0
```

The test feeds the hook the payload shape Claude Code sends and checks the exit code Claude Code acts on.

<figure>
  <img src="{{ '/assets/images/12-testing/hook-test.svg' | relative_url }}" width="100%" alt="Sequence diagram with four lanes: test, payload as Claude Code sends it, hook script, and tool call. In the force-push case the test builds a Bash tool payload, the payload goes to the hook as JSON on standard input with the command git push --force origin main, the hook exits 2 with a reason on standard error, the call is blocked, and the test sees return code 2. In the normal-push case the hook exits 0, the call proceeds, and the test sees return code 0.">
  <figcaption>The behaviour-first hook test, step by step: JSON in on standard input, an exit code out. Made with archify.</figcaption>
</figure>

#### Edge cases this test should grow

| Input | Expected | Why |
|:--|:--|:--|
| `git push --force-with-lease` | Blocked as written (exit 2) | It matches the `--force` pattern; if the policy allows it, change the pattern and pin the new behaviour with a test |
| `git push origin main && rm -rf build` | Allowed by this hook | Another hook handles `rm`; this test pins the scope |
| A payload with no `tool_input.command` | Exit 0 | The hook must not crash on other payload shapes |

## Mocks

A mock replaces a dependency with a stand-in, so the test no longer checks the integration with that dependency.

| Dependency | Mock it? | Instead |
|:--|:--|:--|
| A paid or rate-limited external API | Yes | Replay a recorded real response |
| The network in general | Yes, at the boundary | A local stub server |
| Your own database | Usually no | A temporary test database |
| The filesystem | No | A temporary directory |
| The function under test | Never | The test then checks the mock |

Anthropic's Claude Code best-practices page gives an example prompt that encodes this: "write a test for foo.py covering the edge case where the user is logged out. avoid mocks."

## The value gate

A value gate is a short set of questions a test must answer before it is added. It keeps the function-level tests that protect something and drops the ones that mirror the code. Its parts:

- questions every new test must answer;
- a list of junk patterns to reject;
- the cases where a focused function-level test beats an entry-point test.

### Four questions before adding a test

One open-source project, OpenClaw, keeps a test-audit skill that its agents must apply before adding any test. A missing answer means the test is not added yet.

1. What observable behaviour, invariant or independent contract does it protect?
2. What credible regression makes it fail?
3. Why does existing coverage not already catch that failure?
4. Does it need a production seam (an export, flag or hook) that no production caller needs?

### Junk patterns

The same skill lists patterns to reject:

- expected values computed by the code under test;
- mocks that implement the asserted behaviour;
- tests whose only purpose is to keep test-only exports, globals or wrappers alive.

It also says: "A test that would break under behavior-preserving refactoring is asserting implementation, not behavior."

### Cases for a focused test

| Situation | Better test | Why |
|:--|:--|:--|
| The function is a contract: a file-format parser, a migration, a security check | A focused test on the function | The function is the entry point for that contract |
| A reproduced bug in a pure helper | A focused regression test | It freezes the exact failure |
| Dense pure logic with many cases | A table of inputs and outputs, or a property-based test | Reaching every case through the entry point needs huge fixtures |
| A helper reached only through a large acceptance test | A mutation run, then a focused test where mutants survive | Acceptance tests are often light on assertions |

In one write-up, a file with 100% statement coverage had no unit tests at all: a large acceptance test happened to call every line, and a mutation tool still found 13 surviving mutants. A behaviour-level test can still have weak assertions.

#### Worked example: a property-based test

A property-based test states a rule that holds for every input and lets the tool generate inputs. With Python's Hypothesis library, a CSV round-trip property:

```python
import csv, io
from hypothesis import given, strategies as st

@given(st.lists(st.text(), min_size=1))
def test_csv_row_round_trips(fields):
    out = io.StringIO()
    write_row(out, fields)                    # the function under test
    assert next(csv.reader(io.StringIO(out.getvalue()))) == fields
```

Hypothesis tries commas, quotes, line breaks and empty strings without anyone listing them, which is the bug class in the [regression test example]({{ '/sections/12-testing/regression-tests.html#2-find-the-root-cause' | relative_url }}).

<details markdown="1">
<summary>Aside: evidence that unit tests written by agents can be good</summary>

- A 2026 study found unit tests written by an LLM with retrieved context detected faults in 69% of cases, against 17.2% for general-purpose human-written tests.
- An agent writing property-based tests across 100 popular Python packages filed bug reports that were valid 56% of the time, with patches merged into NumPy.
- In both studies the agent had context and a clear contract to test against. The evidence against per-function tests concerns volume and tests that mirror the code.

</details>

## Test ordering and the spec

### Three orderings compared

| Ordering | What the agent does | Evidence for current models |
|:--|:--|:--|
| Strict TDD | One failing test, just enough code, repeat | No quality or mutation-score gain in one 2026 evaluation, at 3x to 8.5x the tokens |
| Tests first | Writes the tests from the spec, then the code | About the same tokens as no TDD; quality effect unclear |
| Spec first | Writes acceptance criteria before code; tests derive from them | Spec-grounded test generation found 9.8 points more real bugs |

The evaluation's author, Birgitta Böckeler, reported that the agents "still sometimes skipped or faked the red step" and concluded: "I personally have stopped telling my coding agents to write tests first, let alone do TDD."

### Why the spec comes first

When an agent hits a test it cannot pass, someone has to decide whether the test or the code is wrong:

- That decision needs a statement of intent that predates both.
- Without it, the agent's own code becomes the reference, and a loosened test looks reasonable.
- This is the link to [test tampering and oracle integrity]({{ '/sections/12-testing/oracle-integrity.html' | relative_url }}).

Keep the spec in proportion:

| Change | Spec |
|:--|:--|
| A one-line fix | One acceptance criterion in the task description |
| A small feature | Three to five criteria and a "Verify" line |
| A feature that spans components | A plan document with criteria per component |

#### What a proportional spec looks like

```text
Feature: --priority flag on "task new"
AC1  "task new <title> --priority high" stores priority 1; low stores 3; default is 2.
AC2  An unknown value exits 2 with "priority must be high, normal or low".
AC3  "task show <id>" prints the stored priority.
Verify: one entry-point test per AC, run through the real command.
```

Each criterion gets one test, and the last line says how the whole feature is proved. Anthropic's best-practices page recommends this shape for a spec: self-contained, ending "with an end-to-end verification step that proves the feature works."

#### When the spec changes mid-task

1. The agent stops and reports the conflict instead of choosing.
2. The owner updates the criterion.
3. The agent updates the affected test and records the reason (see [Test-edit-reason trailers]({{ '/sections/12-testing/test-edit-detection.html#a-commit-trailer' | relative_url }})).

## Pitfalls

| Pitfall | Example | Fix |
|:--|:--|:--|
| Testing the helper, not the wiring | `is_force_push()` passes while the hook exits 1 | Drive the hook with its real payload |
| A test that never failed | Written after the code, never seen red | Break the code once and watch it fail |
| Mocking your own code | The test asserts what the mock returns | Mock only external, slow or costly dependencies |
| One entry-point test per input variation | Fifty slow CLI runs for a parser | One entry-point test per criterion, a table-driven or property-based test for the variations |
| Spec written after the code | Criteria that describe what the code happens to do | Write criteria before code, even one line |

## From my harness

My testing rules are a path-scoped rule file: it loads only when an agent edits a test file, so it costs nothing on other work. What it says, and what replaced what:

| Rule now | Rule it replaced |
|:--|:--|
| Test the behaviour, not each function | A failure-path test for every public function |
| One test must drive the real caller end to end | No untested public function |
| Coverage is counted in acceptance criteria | 90%+ line coverage |
| A bug fix may add a focused function-level test that freezes the reproduced failure | (no exception existed) |

The old rules produced the wide unit base of the [classic pyramid](#the-classic-testing-pyramid) above, written by agents that mirrored their own code. The rule file is layer 1 of the harness; the bug-fix procedure and the merge gate carry the enforcement (see [Where tests sit in the harness]({{ '/sections/12-testing/#where-tests-sit-in-the-harness' | relative_url }})).

## Sources

- Claude Code best practices: <https://code.claude.com/docs/en/best-practices>, verified 2026-10-08.
- Claude Code hooks reference (exit codes, payload, JSON decisions): <https://code.claude.com/docs/en/hooks>, verified 2026-11-09.
- OpenClaw test-audit skill: <https://github.com/openclaw/openclaw/blob/main/.agents/skills/test-audit/SKILL.md>, verified 2026-10-08.
- Maintainability sensors (13 surviving mutants): <https://martinfowler.com/articles/sensors-for-coding-agents.html>, verified 2026-10-08.
- TDD inside the agent loop: <https://martinfowler.com/articles/exploring-gen-ai/tdd-in-the-agent-loop.html>, verified 2026-10-08.
- Spec-driven test generation (9.8 points): <https://arxiv.org/abs/2608.17177>, verified 2026-10-08.
- LLM vs. human unit tests (69% against 17.2%): <https://arxiv.org/abs/2606.08588>, verified 2026-10-08.
- Agentic property-based testing: <https://arxiv.org/abs/2510.09907>, verified 2026-10-08.
- Write tests, not too many, mostly integration: <https://kentcdodds.com/blog/write-tests>, verified 2026-10-08.
- Hypothesis: <https://hypothesis.readthedocs.io/>, API as commonly documented; not re-verified for this page.

{% include page_nav.html %}
