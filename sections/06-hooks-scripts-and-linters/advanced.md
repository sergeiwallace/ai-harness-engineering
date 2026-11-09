---
title: "Advanced"
parent: "Hooks, scripts and linters"
nav_order: 2
---

# Advanced hooks and scripts topics

As of 2026-11-09.

- Register hooks from one manifest that an install step applies, so a hook is added or removed in one place and a test can check what is registered.
- Before writing a new check, search the repository for a tool that already enforces the rule, using the names the code uses. A new script that repeats an existing check loses the cases the old one learned to handle.
