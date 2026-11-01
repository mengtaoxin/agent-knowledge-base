---
title: Docs that mirror code drift and become noise
tags: [documentation, agents, source-of-truth, drift]
agents: [all]
symptoms:
  - Project docs restate implementation details that already live in code
  - After code changes, docs lag and agents treat the stale prose as authority
  - Maintaining docs means auditing every document after every change
  - Skipping docs forces agents to rediscover the same map from the whole tree
updated: 2026-11-02
---

# Docs that mirror code drift and become noise

## Problem

Documentation helps agents and humans avoid rediscovering the project from scratch. Writing too much of it creates a second description of the same facts. That second description drifts from the code. The drift rarely blocks a capable agent from eventually fixing the work, but it is noise: stale prose competes with the real source, and people start treating outdated text as a rule.

The other extreme is also wrong. Writing almost nothing means every task pays the cost of reading large parts of the tree. The hard part is the scale: keep enough docs that entry points and constraints are cheap to find, without maintaining a prose mirror of the implementation, and without requiring a full documentation audit after every code change.

## Context

Any repository that ships agent-facing guidance (`AGENTS.md`, rules, change-code steps) or human docs that agents also read. The lesson is about which facts belong in markdown versus in code, types, tests, and package scripts. It is not about a specific framework version.

This knowledge base itself follows a related split: entries record transferable symptoms and reasons; they do not mirror application source. Project-local copies of a procedure become a second source of truth and drift when the real procedure changes (see also the note on procedure copies in `large-task-fills-context-and-weakens-output.md`).

## Why

### Why drift hurts even when the agent can still recover

Coding agents weight nearby instructions. A document that looks authoritative will often be followed before the code is checked. When that document restates volatile implementation detail — field lists, exact command chains already in `package.json`, step-by-step mirrors of current modules — any code change that does not update the prose leaves a false local maximum. The agent may still succeed by reading the tree later. Until then, the stale doc is noise, and sometimes a quiet hazard.

The failure is not “documentation exists.” The failure is **a second writable source of truth for facts the code already owns**.

### What a thinner doc set preserves

- **Cheap orientation.** Intent, constraints, and a short map of where to look still save a full-tree search.
- **One durable authority for volatile facts.** Scripts, types, schemas, and tests stay the source; docs point at them instead of copying them.
- **Bounded maintenance.** Updates stay local to the topic that changed. Nobody is forced to sweep every markdown file after every diff.

### Why this approach, not the obvious alternatives

- **Document everything useful for onboarding.** High coverage of current structure ages into a large stale surface. Agents and humans then argue with outdated encyclopedias.
- **Document almost nothing; always read the code.** Correct as a source-of-truth stance, but expensive as a default workflow. Entry points, non-obvious constraints, and “why not the obvious alternative” are repeatedly rediscovered.
- **Require a full documentation audit after every code change.** That policy is accurate in theory and unsustainable in practice. It either gets skipped or burns time disproportionate to the risk of map-level drift.
- **Generate all docs from code on every build.** Useful for API surfaces that already have a single machine-readable source. It does not replace intent, constraints, or failure lessons, and it does not justify hand-written mirrors of the same surface.

The matching property is **volatility versus lookup cost**: write what the code cannot cheaply say; refuse to maintain what the code already says stably.

## What worked

Treat project docs as a thin layer of **intent, constraints, and pointers**, not as a shadow of the tree.

**Write (or keep):**

- Why a constraint exists, and what must not be done
- Entry maps: which files, commands, or docs to open first for a class of change
- Cross-cutting conventions that are hard to infer from one module
- Reusable failure lessons (symptoms, cause, fix) that transfer across projects
- Workflow contracts that are stable: for example “run `verify` after changes,” with details owned by `package.json`

**Do not write (or delete when found):**

- Prose mirrors of APIs, fields, and module inventories that already live in code
- Full command chains that duplicate package scripts
- Architecture encyclopedias that must track every refactor
- Project-local copies of a procedure whose real home is a skill or shared entry

**Maintain without a full-repo doc sweep:**

1. When a change touches a topic, update only the docs that claim authority on that topic.
2. Prefer pointers (“see `package.json` script `verify`”) over copied detail.
3. Accept small drift in map-level prose when the code remains the authority; treat that as noise, not as a release blocker.
4. When a doc and the code disagree, fix the doc or delete the claim — do not leave both standing as peers.

A practical default for agent-friendly repos is: a short always-on rule set, a thin entry doc for how to run and verify, and on-demand lesson entries. Skip the module encyclopedia.

## What did not work

- Expanding docs whenever something was “nice to have written down,” without asking whether code already owned that fact.
- Treating “the agent can still recover from stale docs” as a reason to keep implementation mirrors; recovery does not remove the noise cost.
- Mandating that every pull request re-read every documentation file for drift.

## References

- Related lesson on avoiding a second source of truth for procedures: [large-task-fills-context-and-weakens-output.md](large-task-fills-context-and-weakens-output.md)
