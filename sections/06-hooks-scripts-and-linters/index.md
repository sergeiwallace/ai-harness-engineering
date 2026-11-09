---
title: "Hooks, scripts and linters"
nav_order: 16
has_children: true
has_toc: false
---

# Hooks, scripts and linters

As of 2026-11-09.

Stop 5 on the talk path, about 4 minutes.

- Enforce as much as possible in code, because agents follow written instructions inconsistently as the context fills and work piles up.
- The ladder from soft to hard enforcement: context rule, path rule, skill, hook or guard, test or CI gate.
- One case study: the process-kill rule that was already in the brief, broken anyway, and turned into a hook.
- What guards cost: they block some legitimate work and need maintenance.
- `/hooks` shows every configured hook and where it comes from, read-only.

## Pages for this topic

- [Hooks and scripts in detail]({{ '/sections/06-hooks-scripts-and-linters/detail.html' | relative_url }})
- [Advanced hooks and scripts topics]({{ '/sections/06-hooks-scripts-and-linters/advanced.html' | relative_url }})
