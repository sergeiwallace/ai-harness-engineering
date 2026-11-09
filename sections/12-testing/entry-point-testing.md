---
title: "Testing at the real entry point"
parent: "Testing agent-written code"
nav_order: 1
redirect_from:
  - /sections/12-testing/detail.html
---

{% include page_nav.html %}

# Testing at the real entry point
{: .no_toc }

As of 2026-11-09. Part of [Testing agent-written code]({{ '/sections/12-testing/' | relative_url }}).

A behaviour-first test drives the code through the path a user or another program actually takes, and asserts the outcome the acceptance criteria name. This site uses it as the default test for agent-written code, because an agent cannot make it pass by mirroring its own implementation. This page covers what counts as an entry point, how to write the test, when a function-level test is still the right call, and how to order test writing against the spec.

<details markdown="1">
<summary>Contents of this page</summary>

* TOC
{:toc}

</details>

## The classic testing pyramid

<figure>
  <img src="{{ '/assets/images/12-testing/classic-testing-pyramid.png' | relative_url }}" style="max-width: 381px; width: 100%;" alt="The classic testing pyramid: a wide base labelled Unit, a middle band labelled Integration, and a small top labelled E2E.">
  <figcaption>The classic testing pyramid: most tests at the unit level, few end to end. Image by Mike Wacker, public domain, via Wikimedia Commons: <a href="https://commons.wikimedia.org/wiki/File:Testing_Pyramid.png">File:Testing_Pyramid.png</a>, from the Google Testing Blog post <a href="https://testing.googleblog.com/2015/04/just-say-no-to-more-end-to-end-tests.html">"Just Say No to More End-to-End Tests"</a> (2015).</figcaption>
</figure>

The pyramid was drawn for human teams, where unit tests were cheap to write and end-to-end tests were slow and flaky. Two things change when an agent writes the tests. Writing a test now costs almost nothing, so the base grows without limit, and the agent writes that base by reading its own implementation, so those tests mirror the code instead of the intent. The weight therefore moves up: the default test drives the entry point, integration tests cover how components share state, and the unit base shrinks to the tests that pass the [value gate](#the-value-gate). The cost of end-to-end tests has not gone away (see the flakiness aside on the [overview]({{ '/sections/12-testing/' | relative_url }})), so "drive the entry point" usually means a fast, hermetic run of the real command or route, not a full browser session for every case.

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

## Writing the test

### Step by step

1. Read the acceptance criterion and name the behaviour in one sentence a user would recognise.
2. Find the entry point that behaviour is reached through.
3. Build the smallest real environment it needs: a temporary directory, a test database, a local server.
4. Call the entry point with realistic input.
5. Assert the observable outcome, not the intermediate values.
6. Check that the test fails when you break the behaviour on purpose. A test that passes on broken code is not checking anything.

### Worked example: a hook that blocks a dangerous command

Claude Code can run a hook before every tool call. The hook receives a JSON description of the call on standard input and can block it by exiting with code 2. Suppose an agent writes a hook that blocks `git push --force`.

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
    payload = {"tool_name": "Bash", "tool_input": {"command": command}}
    return subprocess.run(
        ["./tools/block-force-push.sh"],
        input=json.dumps(payload), capture_output=True, text=True,
    )

def test_force_push_is_blocked():
    result = run_hook("git push --force origin main")
    assert result.returncode == 2
    assert "force push" in result.stderr.lower()

def test_normal_push_is_allowed():
    assert run_hook("git push origin main").returncode == 0
```

The test feeds the hook the payload Claude Code would send and checks the exit code Claude Code acts on. If any link in the chain is wrong, it fails.

<figure>
  <img src="{{ '/assets/images/12-testing/hook-test.svg' | relative_url }}" width="100%" alt="Sequence diagram with four lanes: test, payload as Claude Code sends it, hook script, and tool call. In the force-push case the test builds a Bash tool payload, the payload goes to the hook as JSON on standard input with the command git push --force origin main, the hook exits 2 with a reason on standard error, the call is blocked, and the test sees return code 2. In the normal-push case the hook exits 0, the call proceeds, and the test sees return code 0.">
  <figcaption>The behaviour-first hook test, step by step. It exercises the same contract Claude Code relies on: JSON in on standard input, an exit code out. Made with archify.</figcaption>
</figure>

### Mocks

A mock replaces a dependency with a stand-in, so the test no longer checks the integration with that dependency. A common rule is to mock what is slow, costly or outside your control (a paid API, the network) and run the rest for real. When a mock implements the behaviour the test then asserts, the test checks the mock.

> Anthropic's Claude Code best-practices page gives an example prompt that encodes this directly: "write a test for foo.py covering the edge case where the user is logged out. avoid mocks."

## The value gate

This section covers which function-level tests to keep.

### Four questions before adding a test

One open-source project keeps a test-audit skill that its agents must apply before adding any test. A missing answer means the test is not added yet.

1. What observable behaviour, invariant or independent contract does it protect?
2. What credible regression makes it fail?
3. Why does existing coverage not already catch that failure?
4. Does it need a production seam (an export, flag or hook) that no production caller needs?

It also lists junk patterns: expected values computed by the code under test, mocks that implement the asserted behaviour, and tests whose only purpose is to keep test-only exports alive. It also says: "A test that would break under behavior-preserving refactoring is asserting implementation, not behavior."

### Cases for a focused test

| Situation | Better test | Why |
|:--|:--|:--|
| The function is a contract: a file-format parser, a migration, a security check | A focused test on the function | The function is the entry point for that contract |
| A reproduced bug in a pure helper | A focused regression test | It freezes the exact failure |
| Dense pure logic with many cases | A table of inputs and outputs, or a property-based test | Reaching every case through the entry point needs huge fixtures |
| A helper reached only through a large acceptance test | A mutation run, then a focused test where mutants survive | Acceptance tests are often light on assertions |

In one write-up, a file with 100% statement coverage had no unit tests at all: a large acceptance test happened to call every line, and a mutation tool still found 13 surviving mutants. A behaviour-level test can still have weak assertions.

<details markdown="1">
<summary>Aside: evidence that unit tests written by agents can be good</summary>

The evidence against per-function tests concerns volume and tests that mirror the code. A 2026 study found unit tests written by an LLM with retrieved context detected faults in 69% of cases against 17.2% for general-purpose human-written tests. An agent writing property-based tests across 100 popular Python packages filed bug reports that were valid 56% of the time, with patches merged into NumPy. In both studies the agent had context and a clear contract to test against.

</details>

## Test ordering and the spec

### Three orderings compared

| Ordering | What the agent does | Evidence for current models |
|:--|:--|:--|
| Strict TDD | One failing test, just enough code, repeat | No quality or mutation-score gain in one 2026 evaluation, at 3x to 8.5x the tokens |
| Tests first | Writes the tests from the spec, then the code | About the same tokens as no TDD; quality effect unclear |
| Spec first | Writes acceptance criteria before code; tests derive from them | Spec-grounded test generation found 9.8 points more real bugs |

The evaluation's author reported that the agents "still sometimes skipped or faked the red step" and concluded: "I personally have stopped telling my coding agents to write tests first, let alone do TDD."

### Why the spec comes first

When an agent hits a test it cannot pass, someone has to decide whether the test or the code is wrong. That decision needs a statement of intent that predates both. This is the link to [test tampering and oracle integrity]({{ '/sections/12-testing/oracle-integrity.html' | relative_url }}).

Keep the spec in proportion. A one-line acceptance criterion is enough for a small fix. A full plan document is for features that need one.

#### What a proportional spec looks like

```text
Feature: --priority flag on "task new"
AC1  "task new <title> --priority high" stores priority 1; low stores 3; default is 2.
AC2  An unknown value exits 2 with "priority must be high, normal or low".
AC3  "task show <id>" prints the stored priority.
Verify: one entry-point test per AC, run through the real command.
```

Each criterion gets one test, and the last line says how the whole feature is proved. Anthropic's best-practices page recommends this shape for a spec: self-contained, ending "with an end-to-end verification step that proves the feature works."

## From my harness

My testing rules load only when an agent edits a test file. They say: test the behaviour, not each function; one test must drive the real caller end to end; and coverage is counted in acceptance criteria, not lines. These rules replaced an older one that asked for 90% line coverage, no untested public function and a failure-path test for every public function, which is the rule that produced the wide base in the [overview's diagram]({{ '/sections/12-testing/' | relative_url }}). A bug fix is the one place a focused function-level test is always allowed, because it freezes the reproduced failure.

## Sources

- Claude Code best practices: <https://code.claude.com/docs/en/best-practices>, verified 2026-10-08.
- Hooks guide (exit code 2 blocks): <https://code.claude.com/docs/en/hooks-guide>, verified 2026-10-08.
- OpenClaw test-audit skill: <https://github.com/openclaw/openclaw/blob/main/.agents/skills/test-audit/SKILL.md>, verified 2026-10-08.
- Maintainability sensors (13 surviving mutants): <https://martinfowler.com/articles/sensors-for-coding-agents.html>, verified 2026-10-08.
- TDD inside the agent loop: <https://martinfowler.com/articles/exploring-gen-ai/tdd-in-the-agent-loop.html>, verified 2026-10-08.
- Spec-driven test generation (9.8 points): <https://arxiv.org/abs/2608.17177>, verified 2026-10-08.
- LLM vs. human unit tests (69% against 17.2%): <https://arxiv.org/abs/2606.08588>, verified 2026-10-08.
- Agentic property-based testing: <https://arxiv.org/abs/2510.09907>, verified 2026-10-08.
- Write tests, not too many, mostly integration: <https://kentcdodds.com/blog/write-tests>, verified 2026-10-08.

{% include page_nav.html %}
