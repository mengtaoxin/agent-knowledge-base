---
title: Prefer a full unit suite over selective test routing
tags: [testing, unit-tests, tdd, ci, agents]
agents: [all]
symptoms:
  - Agents or humans run only the changed file or module's unit tests and skip the rest of the suite
  - A project invents change-to-package routing rules so each edit runs a subset of automated unit tests
  - Unit tests exist but regressions still land because the full suite rarely runs before merge
  - Engineering effort goes into shrinking which tests run instead of keeping the full suite acceptably fast
updated: 2026-11-05
---

# Prefer a full unit suite over selective test routing

## Problem

Once a project has decided to run automated unit tests, the next choice is which subset to run. Running only the file or module that just changed looks cheaper. Maintaining that habit forces a second system: rules that map “this code changed” to “these packages, folders, or suites must run.” That routing is easy to get wrong, expensive to keep accurate, and often exists only to save wall-clock time on a suite that would finish in minutes if left alone.

The worse outcome is tests that were carefully written and then rarely executed end to end. Partial runs green-wash a change; the full suite is deferred; regressions show up after merge. That inverts the point of writing the tests.

## Context

Applies whenever a coding agent or human runs automated **unit** tests under TDD or as a finish gate: locally, on a shared/cloud machine, or in CI (for example GitHub Actions). This entry is about **suite selection once tests are already going to run**. It is not the finish-workflow / npm script naming contract (format → check → verify); that lives in [inconsistent-format-check-verify-script-names-across-repos.md](inconsistent-format-check-verify-script-names-across-repos.md). Default `verify` there already means static checks plus the full unit suite (`test`); this entry explains why that full unit run should stay preferred over inventing a selective subset.

End-to-end and other heavy suites stay out of scope here unless the repo already folds them into the same gate.

## Why

### Why selective routing looks attractive and still loses

A changed-file or changed-module run only proves what that slice covers. Shared helpers, serializers, config loaders, and cross-package contracts often break outside the edited path. Catching those failures needs either a full suite or a maintained map from paths to dependent suites. The map is judgment cost: which folder owns the tests, which packages import the changed module, what to do for renames and generated code. That cost is paid on every change set, by humans and by agents that invent or follow incomplete rules.

When the full unit suite routinely finishes on the order of **about ten to fifteen minutes**, that machine time is usually cheaper than keeping the routing correct. Buying faster hardware or a slightly larger CI runner to shorten a full run is often a better trade than building auto-routing whose only job is to avoid running tests you already wrote.

### What preferring the full suite preserves

- **One simple rule.** “Run the unit suite” means the whole suite, not a guessed subset.
- **Tests that actually guard merges.** Effort spent writing unit tests is not wasted on paths that never execute before `main`.
- **Lower agent judgment load.** Agents do not need a project-specific “which tests for this diff” policy for ordinary unit coverage.
- **Quality over micro-optimizing runner time.** Shortening a green full run matters less than shipping code that the suite has actually exercised — until the suite itself becomes extreme.

### Why this approach, not the obvious alternatives

- **Always run only the touched file or package.** Fast feedback, weak regression signal. Cross-cutting breaks stay invisible until something larger runs — if it ever does.
- **Build change-impact / package routing for unit tests.** Useful in huge monorepos when a full run is truly pathological. For suites that finish in minutes, the maintenance and miss rate of the router usually outweigh the time saved.
- **Optimize the suite as soon as anyone waits a few minutes.** Premature. Treat roughly **half a day (for example more than about six hours)** of regular full-unit runtime as the threshold where suite design, parallelism, or tooling investment is justified. Tens of minutes is still an acceptable full run for this policy.
- **Rely on CI subset jobs and never require a full unit run before `main`.** Local and PR shortcuts then become the only signal. The floor in this policy is: if a full unit run cannot be the everyday default, it must still run once before code lands on `main`.

## What worked

When automated unit tests are going to run, **prefer the full unit suite** over a changed-file or changed-module subset.

Practical defaults:

1. **Default command:** the project's full unit entry point (for example `npm test` / `npm run test` once, not watch mode), locally or on CI / a shared runner.
2. **Environments:** same preference on a laptop, a cloud or company machine, and GitHub Actions (or equivalent). Do not invent a weaker subset policy for agents than for humans.
3. **Acceptable cost:** a full unit run on the order of about ten to fifteen minutes is fine; do not add selective routing just to shave that down.
4. **When to optimize the suite itself:** when a full unit run regularly takes on the order of half a day (for example more than about six hours). Prefer making the full run faster over teaching every change which subset to run — until that extreme.
5. **Hard floor:** if everyday full runs are impractical, still require **one full unit suite run before merge to `main`**. Partial runs must not be the last gate onto the default branch.
6. **Priority:** code quality and real coverage beat saving a few minutes of unit-runner time. Runner time is secondary, not irrelevant.

Do not treat “we wrote unit tests” as done if the usual path only ever executes a narrow slice of them.

## What did not work

- Maintaining path-to-suite routing whose main benefit was a shorter unit run, while the unrouted full suite would have finished in minutes.
- Treating a green single-file or single-module run as sufficient evidence to merge, then discovering cross-module regressions after landing on `main`.
- Writing a large unit suite and then routinely skipping most of it so the suite became documentation instead of a gate.

## References

- Finish-workflow and npm script names (`test` = full unit suite once; `verify` includes that suite): [inconsistent-format-check-verify-script-names-across-repos.md](inconsistent-format-check-verify-script-names-across-repos.md)
