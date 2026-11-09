---
title: "Taste and spiky capabilities"
parent: "Trends and model changes"
nav_order: 3
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Taste and spiky capabilities

Related: [Building your own harness]({{ '/sections/17-building-a-harness/' | relative_url }}).

Models are now strong at many tasks that look hard and still weak at neighbouring tasks that look easy. They are weakest at taste: judging what is good, as opposed to what is correct. This page covers both ideas, uses the building of this site and its slide deck as a case study, and says what that means for steering agents. The last part gives my own view, labelled as opinion, with a counterpoint.

![The jagged frontier of AI capabilities: a wavy blue line of AI abilities crosses a dashed curve of tasks of equal difficulty, so one task on the curve falls inside the frontier, a neighbouring one falls outside it, and a third sits on it.]({{ '/assets/images/18-trends/jagged-frontier-mollick.jpg' | relative_url }}){: style="width:100%"}

*Placeholder image: the jagged frontier, tasks of equal difficulty landing inside or outside what AI does well. Source: [Ethan Mollick, Centaurs and Cyborgs on the Jagged Frontier](https://www.oneusefulthing.org/p/centaurs-and-cyborgs-on-the-jagged), used with credit; no licence stated.*

## 1. Spiky capabilities

### 1.1 What the term means

A spiky, or jagged, capability profile is one where a model's skill does not follow human ideas of difficulty. In people, skills tend to go together: someone who can write a proof can usually compare two decimals. In models they often do not.

- Andrej Karpathy popularised the term "jagged intelligence" in a July 2024 post; the example widely reported from it is a model that solves hard maths problems yet says 9.11 is larger than 9.9.
  - In his 2025 year in review he wrote that LLMs "display amusingly jagged performance characteristics", being "at the same time a genius polymath and a confused and cognitively challenged grade schooler".
  - His explanation: capability spikes where rewards can be verified, such as maths and code that passes tests.
- The Harvard Business School and Boston Consulting Group field experiment by Dell'Acqua and colleagues (2023) named the same pattern a "jagged technological frontier": tasks of similar apparent difficulty fall on either side of the frontier.
- Ethan Mollick, a co-author of that study, described the frontier in his essay "Centaurs and Cyborgs on the Jagged Frontier": "That wall is the capability of AI, and the further from the center, the harder the task."

### 1.2 What the evidence shows

| Study | What it measured | Finding |
|---|---|---|
| Dell'Acqua et al. (2023), BCG consultants | Realistic consulting tasks with and without GPT-4 | Inside the frontier, consultants with AI completed 12.2% more tasks, 25.1% faster, at 40% higher quality. On a task outside the frontier, where the model gave a convincing wrong answer, consultants without AI were right 84% of the time and those with AI 60 to 70% |
| METR, task time horizons (March 2025, updated January 2026) | The length of software task, in human time, that agents complete with 50% success | The first report found a doubling about every 7 months since 2019. The January 2026 update puts the doubling at about 89 days since 2024, with Claude Opus 4.5 at about 320 minutes (95% interval 170 to 729). The measure is at 50% reliability, on well-specified tasks with automatic scoring |
| METR, time horizons across domains (July 2025) | The same measure on other benchmarks | Visual computer use (OSWorld, WebArena) has horizons about 40 to 100 times shorter than software and reasoning tasks; doubling times range from about 3 to about 20 months by domain |
| DORA report (October 2024) | Software delivery across organisations as AI adoption rose | An estimated 1.5% drop in delivery throughput and 7.2% drop in delivery stability, alongside small gains in documentation quality, code quality and review speed |
| METR, algorithmic against holistic evaluation (August 2025) | 18 real issues, an agent's pull requests scored by tests and then by human review | The agent passed the human-written tests on 38% of runs, and none of the reviewed pull requests was mergeable as-is; reviewers estimated about 42 minutes of work each to make them mergeable |
| METR, experienced open-source developers (July 2025) | 16 developers, 246 real tasks on their own projects, randomly with or without AI tools | Developers expected to be 24% faster with AI and took 19% longer; afterwards they still believed they had been 20% faster |

Together these say three things:

1. Capability is rising fast on a measurable axis.
   - That axis is well-specified tasks with automatic scoring, which is where taste matters least.
2. The profile is spiky across domains as well as within them.
   - Horizons differ by up to two orders of magnitude between kinds of task.
3. Code that passes the tests can still be unfit to merge.
   - In METR's holistic study, the code that passed was still not mergeable as written. The leading failures on runs that failed tests were documentation (89%), lint and formatting (73%) and other code quality (64%).
4. People misjudge which side of the frontier they are on, in the moment and afterwards.

Two of these studies are small: 16 developers in the productivity trial and 18 issues in the holistic evaluation. Read them as strong signals, with that sample size in mind.

![METR's time-horizon chart on a log scale: the length of task, in human time, that each model completes with 50% success, rising from seconds for GPT-2 in 2019 to several hours for Claude Opus 4.5 in late 2025, with wide error bars on recent models.]({{ '/assets/images/18-trends/metr-time-horizon-1-1.png' | relative_url }}){: style="width:100%"}

*Placeholder image: task time horizons from 2019 to 2025, doubling about every 196 days over the whole period. Source: [METR, Time Horizon 1.1](https://metr.org/blog/2026-01-29-time-horizon-1-1/), CC BY as marked on the chart.*

![Our World in Data chart of AI test scores relative to human performance, one line per benchmark, with benchmarks reaching the human level at very different times.]({{ '/assets/images/18-trends/owid-ai-test-scores.png' | relative_url }}){: style="width:100%"}

*Placeholder image: benchmarks reach human-level scores at very different times, another view of an uneven profile. Source: [Our World in Data, Test scores of AI systems on various capabilities relative to human performance](https://ourworldindata.org/grapher/test-scores-ai-capabilities-relative-human-performance), data from Kiela et al. (2023), [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*

![METR chart comparing forecast and observed effects of AI tools on experienced developers: experts and the developers themselves forecast a speedup, while the observed result was a slowdown.]({{ '/assets/images/18-trends/metr-forecast-vs-observed.png' | relative_url }}){: style="width:100%"}

*Placeholder image: forecast speedups against the observed 19% slowdown. Source: [METR, Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/), used with credit.*

### 1.3 Tasks on either side of the frontier

| Models do this well | A neighbouring task they often do poorly |
|---|---|
| Write a function that passes its tests | Choose the abstraction that keeps the module easy to change |
| Fix a failing test | Notice that the test checks the wrong thing |
| Turn data into a chart | Choose the chart type that fits the data |
| Summarise a document accurately | Decide which three points matter to this reader |
| Generate twenty slides from an outline | Make each slide's density fit how it will be presented |
| Follow a stated writing rule | Notice an unstated habit the reader will dislike |
| Solve a hard maths problem | Compare 9.11 and 9.9 (Karpathy's example) |

The right-hand column is mostly judgement: the tasks where the answer depends on what is good for a particular purpose.

## 2. Taste

### 2.1 What taste is

Taste is the ability to judge what is good, as distinct from what is correct.

- Paul Graham's essay "Taste for Makers" argues that taste is real and can be better or worse: "Poof goes the axiom that taste can't be wrong."
- His formula: "The recipe for great work is: very exacting taste, plus the ability to gratify it."
  - Models increasingly have the ability to gratify. The exacting taste is what the person steering brings.

In the work agents do, taste shows up as:

- Code design: which abstraction to introduce, what to leave out, when a simpler design is better.
- Decisions: which option to choose, and when the right answer is not to build the thing at all.
- Visual form: which diagram or chart fits the information, and how much goes on one screen.
- Prose: plain and dense, with no filler, no performed emphasis and no closing aphorism.
- Structure: how to divide a page, a document or a deck so a reader can find and follow it.

### 2.2 Why it is hard for models today

| Reason | How it shows up |
|---|---|
| They produce plausible output | The first answer looks like a good answer, and often is a typical one |
| Their defaults are an average of their training data | Common patterns win over the better fit: a bar chart, a bulleted list, a summary closer |
| Self-critique is weak | Asked to review their own work, models tend to approve it or fix surface issues |
| They rarely refuse the task | They seldom say "this is the wrong thing to build" or "this page should be split" |
| Taste depends on the audience | What is good depends on a reader and a purpose the model sees only partly |
| Feedback on taste is slow and hard to verify | Chris Olah describes research taste, "the ability to chose good problems to work on", as trainable but slow to learn, because feedback on a choice takes months; that is the opposite of the verifiable rewards where Karpathy says models spike |
| Correct output can still be poor work | METR's reviewed pull requests passed tests and still needed about 42 minutes each before they could merge |
| Cheap output can crowd out design | GitClear's analysis of 211 million changed lines found cloned lines rising from 8.3% (2021) to 12.3% (2024) and moved, refactored lines falling from 25% to under 10%; a vendor study with correlational data, so treat it as a signal only |

![METR Sankey diagram of 102 agent runs across 18 tasks: 63 failed the tests, and of the 39 that passed, the 35 reviewers skimmed contained no pull request mergeable without changes.]({{ '/assets/images/18-trends/metr-tests-vs-mergeable.png' | relative_url }}){: style="width:100%"}

*Placeholder image: runs that pass the tests are still not mergeable as written. Source: [METR, Research Update: Algorithmic vs. Holistic Evaluation](https://metr.org/blog/2025-08-12-research-update-towards-reconciling-slowdown-with-time-horizons/), used with credit.*

When ability runs without taste, the result is often what Simon Willison calls slop: "if it's mindlessly generated and thrust upon someone who didn't ask for it, slop is the perfect term for it."

## 3. Case study: building this presentation

This site and its slide deck were built largely by agents over a few days, with me reviewing. These are taste failures the agents made, how each was caught, and what happened next.

| Failure | How it was caught | Encoded in the harness afterwards | Still needs a person |
|---|---|---|---|
| A bar chart for the enforcement ladder, which is a process | My review of the page | A rule in the agents' brief that diagrams show interactions or flows, and a diagram tool that offers flow, sequence and state types | Choosing which form fits each new piece of content |
| Sections written as a single paragraph | My review; then a script that flags any section with no list, table, code, image or callout | The script, run on every page | Deciding what structure fits the content |
| Notes and callouts that restated the text | My review | A rule: keep a callout only if it adds a setting, an edge case, a number or a real warning | Judging whether a note is informative |
| Link text such as "detail", and raw URLs | My review | A rule and a grep: every link has display text, sources go in a References list | Choosing good display text |
| Tables that scrolled sideways on the site | My review of the published site | A site-wide style rule that wraps table cells | Noticing new layout problems |
| Sparse slides | My review of the deck | Nothing yet; a review pass densified them | Judging what one slide can carry while I talk |
| Slop prose: salience sentences, subtraction closers ("X, not Y"), one practice stated as a rule | My review | A writing anti-patterns list the agents check against, with a grep for the patterns a program can find | Tone, and new patterns the list does not cover |
| "Systems thinking" framed as a separate idea from improving the harness | My review | Nothing; the pages were rewritten to present a systems approach, applied to each gap | Whether a framing serves the argument |

Two patterns stand out:

1. Every failure was caught by a person reading the output.
   - No agent flagged any of them in its own report of the work.
2. Some failures turned into checks, and some did not.
   - Structure, links and known prose patterns became scripts and greps.
   - Choosing a visual form, judging a note, and fitting density to a talk stayed with the reviewer.

![Sequence diagram that replaced the bar chart: a never-force-push rule moving from an instruction the agent breaks, to a PreToolUse hook that refuses with a next step, to a permission rule that denies the command.]({{ '/assets/images/17-building-a-harness/enforcement-ladder-sequence.svg' | relative_url }}){: style="width:100%"}

*The flow diagram that replaced the bar chart in the first row of the table, from [Building your own harness]({{ '/sections/17-building-a-harness/' | relative_url }}#3-prefer-a-fix-that-a-program-enforces). Diagram made with archify.*

## 4. What this means for steering agents

1. Supply the taste yourself.
   - State what good looks like for the task: the reader, the purpose, examples you like and dislike.
2. Encode what you can.
   - Linters and greps for known patterns.
   - Templates and stubs for structure.
   - Examples of good output in context files and skills.
   - Review gates, and evaluations that score output against your examples.
   - See [Building your own harness]({{ '/sections/17-building-a-harness/' | relative_url }}#1-a-systems-approach-to-building-a-harness) for where each fix belongs.
3. Accept that some judgement does not encode.
   - Keep a person on decisions where the answer depends on taste, and budget time for that review.
4. Watch the work, and stop it when it drifts.
   - Kent Beck describes the same job in agentic coding: "In augmented coding you care about the code, its complexity, the tests, & their coverage", and "I watched the intermediate results of the genie more carefully, ready to intervene & stop unproductive development."
5. Review the output, and not the agent's report of it.
   - In the case study, every report said the work was done; the problems were visible only in the pages themselves.

## 5. My view: replacement is further off than claimed

This section is my opinion.

Claims that agentic AI will soon replace people in knowledge work are overstated, and the main reason is taste.

- Models now produce large amounts of correct work quickly.
- They still need a person to judge whether that work is good.
  - Whether the design is right.
  - Whether the page reads well and the visual fits the information.
  - Whether the thing should have been built at all.
- The measured progress is on the axis where taste matters least.
  - Well-specified tasks with automatic scoring, at 50% reliability.
- One weak link is enough to keep a person in a job.
  - Mollick puts it as "If you want to understand where AI is headed, don't watch the benchmarks. Watch the bottlenecks." Taste is one of those bottlenecks.
- Building this presentation took days of agent work and constant review.
  - Without that review, the result would have been mostly correct and much weaker.

### 5.1 The counterpoint

| Argument | Evidence |
|---|---|
| Capability is rising quickly | METR's time horizon doubled about every 89 days from 2024, by its January 2026 measure |
| Very long tasks are coming within reach | METR and Epoch AI's preliminary MirrorCode results suggest some weeks-long coding tasks are already possible, on tasks with a clear reference to check against |
| Taste can be partly taught | Rules, examples and evaluations moved several of the case-study failures into checks |
| The frontier moves | The 2023 study used GPT-4; newer models have moved the frontier since, and I expect some of today's taste failures to move inside it too |
| Preference training teaches something taste-like | In OpenAI's InstructGPT work, labellers preferred outputs of a 1.3B-parameter model trained on human preferences to those of the 175B GPT-3; taste expressed as comparisons can be learned, at least the average taste of the raters |
| People are jagged too | The case is that people and models have different shapes of strength, so the combination does better than either |
| People misjudge AI help in both directions | METR's developers thought they were faster when they were slower; the same bias can make people underrate progress |

My answer to the counterpoint: the pace is real, and I expect the taste gap to narrow. It is also the part of the work that is hardest to measure, so it is the part where progress is hardest to see and to trust. For now, I plan on keeping a person in the loop for every decision that depends on judgement.

## References

- Andrej Karpathy's July 2024 post on "jagged intelligence", as reported in [MarkTechPost, Andrej Karpathy coined a new term: jagged intelligence](https://www.marktechpost.com/2024/08/11/andrej-karpathy-coined-a-new-term-jagged-intelligence-understanding-the-inconsistencies-in-advanced-ai/); the original post could not be opened, so the 9.11 example is as reported.
- [Andrej Karpathy, 2025 LLM Year in Review](https://karpathy.bearblog.dev/year-in-review-2025/), 2025-12-19, opened 2026-11-09; source of the quotes.
- [METR, Time Horizon 1.1](https://metr.org/blog/2026-01-29-time-horizon-1-1/), 2026-01-29, opened 2026-11-09.
- [METR, How Does Time Horizon Vary Across Domains?](https://metr.org/blog/2025-07-14-how-does-time-horizon-vary-across-domains/), 2025-07-14, opened 2026-11-09.
- [METR, Research Update: Algorithmic vs. Holistic Evaluation](https://metr.org/blog/2025-08-12-research-update-towards-reconciling-slowdown-with-time-horizons/), 2025-08-13, opened 2026-11-09.
- [METR, MirrorCode: Evidence that AI can already do some weeks-long coding tasks](https://metr.org/blog/2026-04-10-mirrorcode-preliminary-results/), 2026-04-10, summary opened 2026-11-09.
- [Ethan Mollick, The Shape of AI: Jaggedness, Bottlenecks and Salients](https://www.oneusefulthing.org/p/the-shape-of-ai-jaggedness-bottlenecks), 2025-12-20, opened 2026-11-09.
- [Paul Graham, Taste for Makers](https://paulgraham.com/taste.html), 2002, opened 2026-11-09.
- [Chris Olah, Research Taste Exercises](https://colah.github.io/notes/taste/), 2021, opened 2026-11-09.
- [Simon Willison, Slop is the new name for unwanted AI-generated content](https://simonwillison.net/2024/May/8/slop/), 2024-05-08, opened 2026-11-09.
- [GitClear, AI Copilot Code Quality: 2025 Look Back at 12 Months of Data](https://www.gitclear.com/ai_assistant_code_quality_2025_research), opened 2026-11-09.
- [Google Cloud, Highlights from the 10th DORA report](https://cloud.google.com/blog/products/devops-sre/announcing-the-2024-dora-report), 2024-10-22, opened 2026-11-09.
- [Kent Beck, Augmented Coding: Beyond the Vibes](https://newsletter.kentbeck.com/p/augmented-coding-beyond-the-vibes), 2025-06-25, opened 2026-11-09.
- [Ouyang et al., Training language models to follow instructions with human feedback](https://arxiv.org/abs/2203.02155), 2022, opened 2026-11-09.
- [Our World in Data, Test scores of AI systems relative to human performance](https://ourworldindata.org/grapher/test-scores-ai-capabilities-relative-human-performance), CC BY 4.0.
- [Dell'Acqua et al., Navigating the Jagged Technological Frontier, Harvard Business School working paper, 2023](https://www.hbs.edu/faculty/Pages/item.aspx?num=64700); figures confirmed through a search on 2026-11-09, as the paper's pages blocked direct reading.
- [Ethan Mollick, Centaurs and Cyborgs on the Jagged Frontier](https://www.oneusefulthing.org/p/centaurs-and-cyborgs-on-the-jagged), One Useful Thing, 2023-09-16, read 2026-11-09; source of the quote and of the 40% and 84% figures.
- [METR, Measuring AI Ability to Complete Long Tasks](https://metr.org/blog/2025-03-19-measuring-ai-ability-to-complete-long-tasks/), 2025-03-19, read 2026-11-09. METR notes that its figures are now superseded by a newer methodology.
- [METR, Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/), 2025-07-10, read 2026-11-09.

{% include page_nav.html %}
