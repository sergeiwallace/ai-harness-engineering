---
title: "Advanced"
parent: "Skills, commands and plugins"
nav_order: 2
---

# Advanced skills and plugins topics

As of 2026-11-09.

- How a shared library is laid out: one module each for the entry point, options, settings, execution and test helpers. Each skill's tool imports it, so they parse options and load settings the same way.
- Mods need Claude Code 2.1.287 or later and are written as JavaScript or TypeScript handlers.
