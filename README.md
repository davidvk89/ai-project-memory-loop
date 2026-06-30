# ai-project-memory-loop

Supporting files for a one-week case study in using live project documentation as memory during AI-assisted development.

This repository is not a framework, package, or complete methodology. It is the artifact trail behind three companion articles about keeping AI-assisted development bounded, reviewable, and human-led.

## Core idea

AI-assisted development has a context problem.

Coding agents can move quickly, but they only work from the context they are given. A long planning conversation can contain rejected ideas, shorthand, future plans, temporary assumptions, and half-decisions. For a human, that is normal design exploration. For a coding agent, it can become accidental permission.

The workflow documented here uses a lightweight loop:

```text
Task Brief      -> explore the problem
Codex Contract  -> define the bounded implementation
Final Review    -> test, inspect, patch, and update project memory
