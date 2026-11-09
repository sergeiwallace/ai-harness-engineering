---
title: "Talk slides"
nav_order: 2
---

{% include page_nav.html %}

# Talk slides

As of 2026-11-09.

The talk has 13 stops in 42 minutes, then 3 minutes of questions. The slides carry few words, with a diagram where one helps. The pages on this site carry the full text, the examples and the sources.

{% assign slides_url = site.data.slides.url | relative_url %}

[Present the slides]({{ slides_url }}){: .btn .btn-primary target="_blank" rel="noopener" }
[Download the PDF]({{ site.data.slides.pdf | relative_url }}){: .btn }

## Presenting from this site

The slides open in a new tab. These keys work in the deck:

| Key | What it does |
|---|---|
| F | Full screen; Esc leaves it |
| Right arrow, Space | Next slide |
| Left arrow | Previous slide |
| Esc or O | Overview of every slide; click one to jump to it |
| S | Speaker notes in a separate window, with a timer and the next slide |

The PDF has one page per slide, for showing the talk without a browser or for printing.

## How the slides and the pages link

1. Every slide has its own address, counting from 0.
   - `talk/slides/#/17` opens slide 18, [Process-kill guard case study]({{ slides_url }}#/17){:target="_blank" rel="noopener"}.
   - The lists below link every slide this way.
2. Every page on the site links its slide.
   - The header and footer of each page show "Slide:" and the slide title, and open it in a new tab.
   - A section overview and all of its topic pages link the first slide of that section.
   - The landing page links the title slide.
3. Every main slide links back to the site.
   - The slide footer has a "Read more" link to the section overview, opened in a new tab.
   - It also links the section's appendix slide, for questions.
4. The appendix slides each summarise one section and link its overview page.

## Talk timing

| Stop | Minutes | Slides | Section pages |
|---:|---:|---:|---|
| 1 | 2 | 4 | Introduction |
| 2 | 2 | 2 | Agentic AI fundamentals |
| 3 | 3 | 3 | AI harnesses |
| 4 | 6 | 6 | Claude Code and Codex, context files, skills, templates |
| 5 | 4 | 4 | Hooks, scripts and linters |
| 6 | 4 | 3 | Delegation |
| 7 | 4 | 3 | Session coordination, orchestration |
| 8 | 2 | 1 | Context window and memory |
| 9 | 4 | 4 | Testing, code review |
| 10 | 3 | 2 | Human gates and alignment |
| 11 | 2 | 2 | Logging and evaluation |
| 12 | 5 | 4 | Building your own harness |
| 13 | 1 | 2 | Trends and model changes |

The stops add up to 42 minutes for 40 main slides. Questions take the last 3 minutes, using the 21 appendix slides.

## Main slides

### Stop 1, 2 minutes

Section page: [Introduction]({{ '/' | relative_url }})

1. [Agentic AI Workflows & Harness Engineering]({{ slides_url }}#/0){:target="_blank" rel="noopener"}
2. [Audience and scope]({{ slides_url }}#/1){:target="_blank" rel="noopener"}
3. [The conductor model]({{ slides_url }}#/2){:target="_blank" rel="noopener"}
4. [Talk route]({{ slides_url }}#/3){:target="_blank" rel="noopener"}

### Stop 2, 2 minutes

Section page: [Agentic AI fundamentals]({{ '/sections/01-agentic-ai-fundamentals/' | relative_url }})

1. [The agent loop]({{ slides_url }}#/4){:target="_blank" rel="noopener"}
2. [Context window and sessions]({{ slides_url }}#/5){:target="_blank" rel="noopener"}

### Stop 3, 3 minutes

Section page: [AI harnesses]({{ '/sections/02-ai-harnesses/' | relative_url }})

1. [Parts of a harness]({{ slides_url }}#/6){:target="_blank" rel="noopener"}
2. [A systems approach to harness engineering]({{ slides_url }}#/7){:target="_blank" rel="noopener"}
3. [Harness cost and effect]({{ slides_url }}#/8){:target="_blank" rel="noopener"}

### Stop 4, 6 minutes

Section pages: [Claude Code and Codex basics]({{ '/sections/03-claude-code-and-codex/' | relative_url }}), [Context files]({{ '/sections/04-context-files/' | relative_url }}), [Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }}), [Templates, prompt stubs and injected prompts]({{ '/sections/07-templates-and-stubs/' | relative_url }})

1. [The building blocks]({{ slides_url }}#/9){:target="_blank" rel="noopener"}
2. [Claude Code and Codex]({{ slides_url }}#/10){:target="_blank" rel="noopener"}
3. [Native commands]({{ slides_url }}#/11){:target="_blank" rel="noopener"}
4. [Context files]({{ slides_url }}#/12){:target="_blank" rel="noopener"}
5. [Skills and plugins]({{ slides_url }}#/13){:target="_blank" rel="noopener"}
6. [Templates and briefs]({{ slides_url }}#/14){:target="_blank" rel="noopener"}

### Stop 5, 4 minutes

Section page: [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }})

1. [Enforcing rules in code]({{ slides_url }}#/15){:target="_blank" rel="noopener"}
2. [Enforcement options]({{ slides_url }}#/16){:target="_blank" rel="noopener"}
3. [Process-kill guard case study]({{ slides_url }}#/17){:target="_blank" rel="noopener"}
4. [Cost of guards]({{ slides_url }}#/18){:target="_blank" rel="noopener"}

### Stop 6, 4 minutes

Section page: [Agent profiles, sub-agents and delegation]({{ '/sections/08-delegation/' | relative_url }})

1. [Sub-agents and agent profiles]({{ slides_url }}#/19){:target="_blank" rel="noopener"}
2. [Delegation with worktrees]({{ slides_url }}#/20){:target="_blank" rel="noopener"}
3. [Checking delegated work]({{ slides_url }}#/21){:target="_blank" rel="noopener"}

### Stop 7, 4 minutes

Section pages: [Coordinating several sessions]({{ '/sections/09-session-coordination/' | relative_url }}), [Orchestration and workflows]({{ '/sections/10-orchestration/' | relative_url }})

1. [Coordinating several sessions]({{ slides_url }}#/22){:target="_blank" rel="noopener"}
2. [Two kinds of orchestration]({{ slides_url }}#/23){:target="_blank" rel="noopener"}
3. [Goals and loops]({{ slides_url }}#/24){:target="_blank" rel="noopener"}

### Stop 8, 2 minutes

Section page: [Context window and memory]({{ '/sections/11-context-window/' | relative_url }})

1. [Context window management]({{ slides_url }}#/25){:target="_blank" rel="noopener"}

### Stop 9, 4 minutes

Section pages: [Testing agent-written code]({{ '/sections/12-testing/' | relative_url }}), [Code review]({{ '/sections/13-code-review/' | relative_url }})

1. [Testing agent-written code]({{ slides_url }}#/26){:target="_blank" rel="noopener"}
2. [Path of one change]({{ slides_url }}#/27){:target="_blank" rel="noopener"}
3. [Review practices]({{ slides_url }}#/28){:target="_blank" rel="noopener"}
4. [Open-ended review]({{ slides_url }}#/29){:target="_blank" rel="noopener"}

### Stop 10, 3 minutes

Section page: [Human gates, decisions and alignment]({{ '/sections/14-human-gates/' | relative_url }})

1. [Human gates]({{ slides_url }}#/30){:target="_blank" rel="noopener"}
2. [Alignment with the original request]({{ slides_url }}#/31){:target="_blank" rel="noopener"}

### Stop 11, 2 minutes

Section page: [Logging, observability and evaluation]({{ '/sections/16-logging-and-evaluation/' | relative_url }})

1. [Evaluating harness changes]({{ slides_url }}#/32){:target="_blank" rel="noopener"}
2. [Logging and evaluation]({{ slides_url }}#/33){:target="_blank" rel="noopener"}

### Stop 12, 5 minutes

Section page: [Building your own harness]({{ '/sections/17-building-a-harness/' | relative_url }})

1. [Building your own harness]({{ slides_url }}#/34){:target="_blank" rel="noopener"}
2. [Agents improving the harness]({{ slides_url }}#/35){:target="_blank" rel="noopener"}
3. [Complexity creep]({{ slides_url }}#/36){:target="_blank" rel="noopener"}
4. [A possible first week]({{ slides_url }}#/37){:target="_blank" rel="noopener"}

### Stop 13, 1 minute

Section page: [Trends and model changes]({{ '/sections/18-trends/' | relative_url }})

1. [Model changes and trends]({{ slides_url }}#/38){:target="_blank" rel="noopener"}
2. [Questions and further reading]({{ slides_url }}#/39){:target="_blank" rel="noopener"}

## Appendix slides

One backup slide per section, used during questions. The [appendix index slide]({{ slides_url }}#/40){:target="_blank" rel="noopener"} links all of them. Each line below gives the slide, then the section overview it summarises.

1. [Introduction detail]({{ slides_url }}#/41){:target="_blank" rel="noopener"}, overview: [Introduction]({{ '/' | relative_url }})
2. [Agentic AI fundamentals detail]({{ slides_url }}#/42){:target="_blank" rel="noopener"}, overview: [Agentic AI fundamentals]({{ '/sections/01-agentic-ai-fundamentals/' | relative_url }})
3. [AI harnesses detail]({{ slides_url }}#/43){:target="_blank" rel="noopener"}, overview: [AI harnesses]({{ '/sections/02-ai-harnesses/' | relative_url }})
4. [Claude Code and Codex basics detail]({{ slides_url }}#/44){:target="_blank" rel="noopener"}, overview: [Claude Code and Codex basics]({{ '/sections/03-claude-code-and-codex/' | relative_url }})
5. [Context files detail]({{ slides_url }}#/45){:target="_blank" rel="noopener"}, overview: [Context files]({{ '/sections/04-context-files/' | relative_url }})
6. [Skills, commands and plugins detail]({{ slides_url }}#/46){:target="_blank" rel="noopener"}, overview: [Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }})
7. [Hooks, scripts and linters detail]({{ slides_url }}#/47){:target="_blank" rel="noopener"}, overview: [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }})
8. [Templates and stubs detail]({{ slides_url }}#/48){:target="_blank" rel="noopener"}, overview: [Templates, prompt stubs and injected prompts]({{ '/sections/07-templates-and-stubs/' | relative_url }})
9. [Agent profiles, sub-agents and delegation detail]({{ slides_url }}#/49){:target="_blank" rel="noopener"}, overview: [Agent profiles, sub-agents and delegation]({{ '/sections/08-delegation/' | relative_url }})
10. [Coordinating several sessions detail]({{ slides_url }}#/50){:target="_blank" rel="noopener"}, overview: [Coordinating several sessions]({{ '/sections/09-session-coordination/' | relative_url }})
11. [Orchestration and workflows detail]({{ slides_url }}#/51){:target="_blank" rel="noopener"}, overview: [Orchestration and workflows]({{ '/sections/10-orchestration/' | relative_url }})
12. [Context window and memory detail]({{ slides_url }}#/52){:target="_blank" rel="noopener"}, overview: [Context window and memory]({{ '/sections/11-context-window/' | relative_url }})
13. [Testing agent-written code detail]({{ slides_url }}#/53){:target="_blank" rel="noopener"}, overview: [Testing agent-written code]({{ '/sections/12-testing/' | relative_url }})
14. [Code review detail]({{ slides_url }}#/54){:target="_blank" rel="noopener"}, overview: [Code review]({{ '/sections/13-code-review/' | relative_url }})
15. [Human gates, decisions and alignment detail]({{ slides_url }}#/55){:target="_blank" rel="noopener"}, overview: [Human gates, decisions and alignment]({{ '/sections/14-human-gates/' | relative_url }})
16. [Browser automation detail]({{ slides_url }}#/56){:target="_blank" rel="noopener"}, overview: [Browser automation and computer use]({{ '/sections/15-browser-automation/' | relative_url }})
17. [Logging, observability and evaluation detail]({{ slides_url }}#/57){:target="_blank" rel="noopener"}, overview: [Logging, observability and evaluation]({{ '/sections/16-logging-and-evaluation/' | relative_url }})
18. [Building your own harness detail]({{ slides_url }}#/58){:target="_blank" rel="noopener"}, overview: [Building your own harness]({{ '/sections/17-building-a-harness/' | relative_url }})
19. [Trends and model changes detail]({{ slides_url }}#/59){:target="_blank" rel="noopener"}, overview: [Trends and model changes]({{ '/sections/18-trends/' | relative_url }})
20. [Chat, desktop and web clients detail]({{ slides_url }}#/60){:target="_blank" rel="noopener"}, overview: [Chat, desktop and web clients]({{ '/sections/19-chat-and-desktop-clients/' | relative_url }})

{% include page_nav.html %}
