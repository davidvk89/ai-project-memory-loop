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
```

The project memory files are updated as part of the work, not someday after the work. That gives the next task a cleaner starting point.

## Companion articles

These files support a three-part field report from my first serious week building with AI assistance:

1. **Vibe Coding Done Right**  
   How I used spec-driven AI-assisted development as a solo developer.

2. **Documentation as Project Memory in AI-Assisted Development**  
   Why live documentation became a control mechanism instead of an afterthought.

3. **Defensive AI Engineering: Hardening the Project Memory Loop**  
   A skeptical look at documentation debt, context isolation, AI verification, and scaling limits.

Add the published links here:

```md
- [Vibe Coding Done Right](TODO)
- [Documentation as Project Memory in AI-Assisted Development](TODO)
- [Defensive AI Engineering: Hardening the Project Memory Loop](TODO)
```

## What this repository contains

### Project memory files

These are the live documentation files used as operational project memory during the case study:

- `DEV_ARCHITECTURE.md` — architecture notes and system boundaries
- `DEV_DATA_SHAPES.md` — stored data shapes and record expectations
- `DEV_API_CONTRACTS.md` — API behavior and response contracts
- `TECHNICAL_DEBT.md` — deferred work, known compromises, and non-current cleanup

The point is not that these exact four files are universal. The point is that the project had a small, explicit memory surface that could be updated, reviewed, and reused.

### Codex Contract sequence

The included contracts are consecutive examples from the same development phase:

- `cp-0011-text.txt` — Thread Context Controller v0
- `cp-0012.txt` — Messages Side Channel v0
- `cp-0012b.txt` — narrow hotfix for lenient placeholder restore behavior
- `CP-0013.txt` — deprecate legacy Enforcement data and add scoped Enforcement v0

Together, they show the workflow operating across multiple tasks rather than as a single isolated prompt.

A contract is not a brainstorm. It is the cleaned-up instruction set given to the coding agent. The important parts are scope, non-goals, preserved behavior, acceptance criteria, verification steps, documentation updates, and final report requirements.

### Prototype context

- `test_project_README.md` — context from the small local prototype used during the case study
- `Documentation as Project Memory in AI-Assisted Development (1).pdf` — PDF copy of the documentation/project-memory article

The full application source is intentionally not the focus of this repository. This repo is about the workflow artifacts and project-memory pattern.

## Suggested reading order

1. Read the companion articles.
2. Open the four project memory files.
3. Read `cp-0011-text.txt`, `cp-0012.txt`, `cp-0012b.txt`, and `CP-0013.txt` in order.
4. Notice how each contract narrows the implementation boundary.
5. Compare the contracts against the memory files to see how implementation work and documentation updates are tied together.

## What to look for

The useful pattern is not the file names or the exact tools. Look for the control loop:

- planning happens before implementation;
- the implementation context is bounded;
- non-goals are written down;
- documentation updates are required by the contract;
- human review remains the final gate;
- deferred work is preserved instead of accidentally implemented;
- the next task starts from current project memory instead of stale chat history.

## What this is not

This repository is not:

- a reusable software package;
- a finished AI development framework;
- a claim that this workflow scales unchanged to large teams;
- a replacement for human testing or judgment;
- a recommendation that every project needs these exact documents.

It is a case-study artifact set from one week of practical AI-assisted development.

## Why this matters

The result was not perfect AI coding.

The result was reviewable AI coding.

That distinction is the point. A working change can still be rejected if it violates scope. A deferred idea can be preserved without being implemented too early. A future task can begin from documented project state instead of whatever the human or chat history happens to remember.

The transferable pattern is:

```text
Explore in conversation.
Build from a contract.
Preserve the result in documentation.
Reuse that documentation as context for the next task.
```

## Scope and limitations

This case study came from a solo developer working on a small local prototype with a Node backend, static browser frontend, local JSON persistence, rollback support, validation gates, and an LLM gateway compatible with local or OpenAI-style providers.

That matters. The workflow is intentionally modest. It worked in this environment because the documentation surface was small enough to review and reuse. Larger teams, larger repositories, and parallel branches would need additional tooling around documentation slicing, validation, and merge conflict management.

The underlying principle still seems useful: AI-assisted development becomes safer when the project has an explicit memory surface and the coding agent receives a bounded implementation contract.
