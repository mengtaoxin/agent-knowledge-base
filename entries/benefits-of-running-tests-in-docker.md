---
title: Benefits of running automated tests in Docker
tags: [docker, testing, unit-tests, e2e, playwright, ci]
agents: [all]
symptoms:
  - Local or agent-driven unit and e2e runs depend on the host machine's packages, browsers, and caches
  - Playwright or other browser tooling re-downloads or fails under a coding-agent sandbox
  - Dev Containers feel too heavy when only the test run needs isolation
  - Local test environment drifts from CI container runners
updated: 2026-11-08
---

# Benefits of running automated tests in Docker

## Problem

Automated tests — unit suites and e2e alike — often run directly on the developer's machine or inside a coding-agent shell. That couples the suite to whatever the host happens to have: Node, system libraries, browser binaries, and whatever cache paths the agent sandbox injected. The failure that pushed this pattern was Playwright under Cursor's Agent sandbox: browsers were re-downloaded every run, and bypassing the sandbox cache redirection was awkward. The deeper problem is not Playwright itself. It is that the test environment is not a fixed, isolated place.

A related trap is to solve isolation by putting *everything* in a Dev Container. Edit, format, lint, and test then all require the container. That binds day-to-day work too tightly to one environment when only the test run needed the fixed box.

## Context

- Any project with automated unit tests, e2e tests, or both
- Local machines, coding-agent shells (including Cursor), and cloud runners
- Docker as the isolation boundary for the *test* process, not necessarily for the whole workspace
- Cross-project convention: a single environment variable `RUN_TESTS_IN_CONTAINER`
  - `0` (default): run tests on the host
  - `1`: run tests inside a Docker container
- Only the values `0` and `1` are supported
- Originating pain: Playwright cache and launch friction in Cursor's sandbox; the pattern still applies to projects that never use Playwright

## Why

### Why host-bound test runs keep hurting

A host-local suite inherits the host's OS, package set, browser install, and agent-injected environment variables. Coding-agent sandboxes can empty or redirect caches (for example Playwright's browser path). Cloud CI already runs in containers; a laptop that runs the same suite on bare metal is a second, drifting environment. Every “works on my machine / fails in CI” story is a symptom of that split.

### What running tests in Docker preserves

- **A fixed test environment.** Dependencies and browsers live in the image and volumes the suite actually uses, not in whatever the current shell inherited.
- **Isolation from day-to-day host tools.** File edit, format, and lint stay on the host. Only the configured test command opts into the container when `RUN_TESTS_IN_CONTAINER=1`.
- **One switch across projects.** The same `RUN_TESTS_IN_CONTAINER` convention decides where unit and e2e runs execute, so agents and humans do not invent per-repo flags.
- **Parity with how CI and cloud agents already work.** GitHub Actions and similar runners execute automation in containers. Cursor Cloud and other cloud agent environments are container-backed too. Running the local suite in Docker makes “local” and “remote” the same shape of environment.

### Why this approach, not the alternatives

- **Bypass agent sandbox quirks for one tool (for example rewrite `PLAYWRIGHT_BROWSERS_PATH`).** Fixes one symptom on one product. The next tool or the next sandbox rule needs another bypass. Containerizing the test run removes the host/agent surface for the whole suite.
- **Dev Containers for the entire workspace.** Useful when the team wants every command inside the container. Too heavy when the goal is only a stable test runner: every edit and lint becomes container-bound, and leaving that model later is costly. Prefer an opt-in test path via `RUN_TESTS_IN_CONTAINER`.
- **Always run on the host.** Fine as the default (`0`) for fast feedback when the host is enough. It does not give isolation or CI-shaped parity when those matter.
- **Treat cache design as the main lesson.** Caches matter for wall-clock time, but they are not the thesis. Tests in Docker remain valid without warm caches; they are just slow. The lesson is *where* the suite runs.

### Author's path and judgment

The first impulse was to document how to work around Playwright under Cursor's sandbox. That entry was deleted on purpose. Following the sandbox fight far enough made a cleaner rule visible: put the whole automated suite in a container. Even without Playwright, the isolation is worth copying. Matching mainstream CI — tests already run in cloud containers — made the local story feel consistent rather than special-cased. Dev Containers were considered and rejected for everyday use because they pull format, lint, and editing into the container; the preference is to keep those on the host and only flip the test runner when configured.

## What worked

1. Put **all** automated tests that the project treats as the suite — unit and e2e — behind one runner path that can execute inside Docker.
2. Gate that path with `RUN_TESTS_IN_CONTAINER`:
   - unset or `0` → host
   - `1` → container
3. Keep non-test workflows on the host (edit, format, lint). Do not require a Dev Container for ordinary development.
4. When using the container path, arrange **persistent caches via Docker volumes** (or equivalent mounts) so repeated runs do not redo expensive work from scratch. Exact mounts depend on the stack; typical examples include npm's cache or `node_modules`, Playwright browser downloads, and apt/package manager state on Debian/Ubuntu-based images. Image layer cache remains the baseline. Without those, the suite still runs correctly — it is only much slower.
5. Reuse the same `RUN_TESTS_IN_CONTAINER` name across repositories so agents and scripts share one convention.

Minimal illustration of the switch (shape only; wiring is project-specific):

```bash
# Host (default)
npm test

# Container
RUN_TESTS_IN_CONTAINER=1 npm test
```

## What did not work

- Documenting and maintaining a Cursor-specific Playwright host-cache bypass (`PLAYWRIGHT_BROWSERS_PATH` rewrites, `test:e2e:local`, Seatbelt/`required_permissions: ["all"]` combinations). It papered over one agent sandbox instead of giving the suite a stable environment.
- Adopting Dev Containers so every command runs inside Docker. Isolation for tests came with an unwanted requirement that day-to-day editing and static checks live in the container too.

## References

- Docker volumes for durable data across container restarts: [Volumes](https://docs.docker.com/engine/storage/volumes/)
- Playwright browsers and install locations (when the suite uses Playwright): [Browsers](https://playwright.dev/docs/browsers)
- Related suite-selection policy: [prefer-full-unit-suite-over-selective-test-routing.md](prefer-full-unit-suite-over-selective-test-routing.md)
