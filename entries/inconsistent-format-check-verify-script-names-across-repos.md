---
title: Inconsistent format, check, and verify script names across repos
tags: [javascript, typescript, node.js, npm, scripts, format, check, verify, agents]
agents: [all]
symptoms:
  - One repo uses check as the finish gate and another uses verify
  - format is named fmt or format, and type-check is named typecheck or type-check
  - An aggregate command writes files (auto-format) while sounding like a read-only check
  - An agent does not know whether to run check, verify, or a chain of atomic scripts
  - Finish steps differ by project even when the intended gate is the same
updated: 2026-10-31
---

# Inconsistent format, check, and verify script names across repos

## Problem

Across several personal repositories, the same finish workflow — format, lint, type-check, unit tests, sometimes e2e — is exposed under different npm script names and different meanings. One project’s `check` includes tests; another’s `verify` auto-formats; a third has no aggregate and documents a hand-built chain. Humans and coding agents waste time rediscovering the gate, and a command that sounds read-only sometimes rewrites the tree.

## Context

**Ecosystem:** JavaScript / TypeScript / Node.js projects that expose the gate through npm (or compatible) scripts in `package.json`. The names and read-only rules in this entry are for that ecosystem; they are not a general task-runner contract for unrelated stacks.

Applies to multi-repo setups that share a similar TypeScript / oxlint / oxfmt (or equivalent) toolchain. The lesson is the shared contract for script names and agent finish steps. Per-repo details (workspace filters, extra validators, how `verify` is implemented) stay in that repo’s `package.json`, `docs/commands.md`, or `docs/change-code-steps.md`.

## Why

### Why the failure happens

Aggregate script names carry meaning. `check` and `verify` both suggest “assert the tree is good,” not “mutate then assert.” Abbreviated names (`fmt`, `typecheck`) and swapped aggregates (`check` that runs tests in one repo, `verify` that only lints in another) break that expectation. Agents follow the host project’s docs or invent a chain; without a shared contract they pick the wrong gate or skip tests.

### What the fix preserves

A small, stable vocabulary so every repo answers the same questions the same way: what writes files, what is a fast static gate, what is the finish gate, and what stays optional (e2e, production build). Implementation can stay thin (`&&` or a small script); the names and read-only rules stay aligned.

### Why this approach

- **Shared naming contract (this entry)** gives agents and humans one place to look before aligning a repo. Project-local copies of the same table drift.
- **Putting auto-format inside `verify`** is convenient but blurs “verify = read-only.” Format stays a separate write step; `verify` only asserts.
- **A third aggregate name** (`ci`, `gate`, `validate` as the finish command) adds another synonym without adding meaning. Stick to `check` and `verify`.
- **Forcing identical `verify.mts` implementations** is unnecessary. Small repos can use a one-line script chain; large monorepos can be smarter. The external contract matters more than the runner.

## What worked

Use this contract in every JavaScript / TypeScript / Node.js repo that has format / lint / type-check / test tooling exposed as npm scripts.

### Script names

| Script | Writes files? | Meaning |
| --- | --- | --- |
| `format` | Yes | Format in place. Prefer the full word; do not use `fmt`. |
| `format:check` | No | Fail if any file is unformatted. |
| `lint` | No | Lint. |
| `type-check` | No | Typecheck only (`tsc --noEmit` or project equivalent). Prefer the hyphenated name over `typecheck`. |
| `test` | No | Full unit suite, once (not watch). Watch mode is `test:watch` if needed. |
| `test:e2e` | No | End-to-end suite. Prefer this name over a bare `e2e`. |
| `check` | No | Fast static gate: `format:check` → `lint` → `type-check`. No unit tests, no e2e, no production build (unless the project has no separate type-check and must use build for that role). |
| `verify` | No | Finish gate for a change set: everything in `check`, then `test`. Read-only end to end. |

Optional convenience that **does** write, and must not be named `verify` or `check`:

- `fix` (or documented `npm run format && npm run verify`): run `format`, then `verify`.

### Defaults for heavy steps

- **e2e** is not part of default `verify`. Offer `verify --e2e`, or document a separate `test:e2e` when the change needs that layer.
- **Production `build`** is not part of default `verify`. Run it when the change touches routing, bundler config, Server Components, or other cases the repo documents.

Project-specific validators (config schema checks, doc link checks, and so on) may hang off `verify` when they are part of “this change is done,” as long as they stay read-only and the repo documents them.

### Agent finish steps

When finishing a code change in a host project:

1. Discover scripts from `package.json`, `AGENTS.md`, and `docs/commands.md` or `docs/change-code-steps.md`.
2. While iterating: use focused tests; use `npm run check` (or the atomic static trio) when a static pass is enough.
3. Before calling the change done: if the tree needs formatting, run `npm run format`. Then run `npm run verify`.
4. If there is no `verify`, compose the read-only chain: `format:check` → `lint` → `type-check` → `test` (plus e2e / build only when this repo’s docs or the change scope require them).
5. Do not invent a third aggregate finish name. Prefer `verify` for the finish gate and `check` for the static-only gate.

Success output for the finish command should stay easy to scan (for example a single line like `✔ verify passed`) when the repo already prints a summary.

### Aligning an existing repo

1. Rename `fmt` / `fmt:check` → `format` / `format:check` (temporary aliases are fine briefly).
2. Normalize `typecheck` → `type-check` where practical.
3. Make `check` the read-only static aggregate; make `verify` the read-only finish aggregate (static + unit tests).
4. Remove auto-format from inside `verify` / `check`. Keep write behavior on `format` or an explicitly named `fix`.
5. Point `AGENTS.md` and change-code steps at `format` (when needed) then `verify`, and link this entry or restate the same table in `docs/commands.md`.

## What did not work

- Using `check` in one repo to mean “static only” and in another to mean “full finish including tests.”
- Putting writable `format` inside `verify` so one command both rewrites and validates; the name stops matching the behavior.
- Leaving only a documented manual chain (`format && lint && type-check && test`) with no `verify`, so every agent reconstructs the gate differently.
- Abbreviating to `fmt` in most repos while one repo already used `format`, which reintroduced the naming split.

## References

- Host project docs such as `docs/commands.md` and `docs/change-code-steps.md` when present — they own repo-specific flags and extra steps; this entry owns the shared names and read-only rules.
