---
title: Cursor Agent re-downloads Playwright browsers every run
tags: [playwright, cursor, sandbox]
agents: [cursor]
symptoms:
  - Playwright installs or downloads Chromium on every agent run
  - PLAYWRIGHT_BROWSERS_PATH points at cursor-sandbox-cache
  - Browser missing after a previous successful install in the same project
  - Executable doesn't exist at .../chrome-headless-shell-mac-x64/...
  - bootstrap_check_in ... Permission denied when launching Chromium
updated: 2026-10-26
---

# Cursor Agent re-downloads Playwright browsers every run

## Problem

When a Cursor Agent runs Playwright tests, each session downloads browser binaries again even though the host already has them under the normal Playwright cache. The download lands in a per-session directory that disappears when the chat ends, so the next run repeats the install.

Cursor injects `PLAYWRIGHT_BROWSERS_PATH` into the Agent shell. The value points at an empty path under `cursor-sandbox-cache`. Playwright treats that empty directory as the browser registry, so it reinstalls. Leaving the sandbox alone does not clear the variable: child processes still inherit it.

## Context

- Cursor Desktop Agent with the default command sandbox
- Playwright / `@playwright/test` (any recent version that honors `PLAYWRIGHT_BROWSERS_PATH`)
- Observed on macOS; default host cache is `~/Library/Caches/ms-playwright`
- This behavior is Cursor-specific. Other coding agents that do not inject `PLAYWRIGHT_BROWSERS_PATH` or a Seatbelt sandbox do not need this entry

Two separate failures stack:

1. **Wrong registry directory.** Even with `required_permissions: ["all"]`, `PLAYWRIGHT_BROWSERS_PATH` still points at the empty sandbox cache path.
2. **Sandbox cannot run the host browsers.** Inside the sandbox, `os.cpus()` is empty, so Playwright may pick the wrong host platform (for example `mac-x64` on Apple Silicon). Chromium also fails to start (`bootstrap_check_in ... Permission denied`), and writing a fresh install into the host cache is blocked.

## Why

The re-download and the launch failure come from two independent injections, so clearing only one of them leaves the other intact.

Cursor sets `PLAYWRIGHT_BROWSERS_PATH` to an empty directory under `cursor-sandbox-cache`. Playwright reads that variable when `playwright-core` loads and treats the directory as its browser registry. The install therefore lands in a path that disappears when the chat ends, and the next run repeats it. Unsetting the variable in the process that starts Node, before any Playwright import, makes Playwright use its platform default cache, which the host already populated. Deleting the variable later, inside config that imports Playwright first, is too late: the registry path is already fixed.

The sandbox is a separate cause. With the variable unset, Playwright looks at the host cache, but inside the sandbox `os.cpus()` is empty, Chromium's bootstrap is denied, and writing a new install into the host cache is blocked. Running the npm script with the sandbox disabled lets CPU detection and Chromium use that host cache. Disabling the sandbox without unsetting the variable leaves the empty registry path in the environment, so Playwright still downloads.

Broader workarounds miss one of those two causes. Turning off the Agent sandbox for every command is wider than this failure. Adding the host cache to sandbox write paths, or relying on a shared build cache, does not override `PLAYWRIGHT_BROWSERS_PATH`. There is no Cursor setting that runs only Playwright outside the sandbox, so the durable form is a project rule for `npm run` plus `env -u` on the test script.

## What worked

Apply both steps. Either one alone is not enough.

### 1. Unset the injected path before Playwright starts

Playwright reads `PLAYWRIGHT_BROWSERS_PATH` when `playwright-core` loads and keeps that value. Unset it in the process that starts Node, before any Playwright import. In `package.json`:

```json
"test:e2e": "env -u PLAYWRIGHT_BROWSERS_PATH playwright test"
```

After unset, Playwright uses its platform default cache (on macOS, `~/Library/Caches/ms-playwright`). Do not hard-code that path; defaults differ by OS.

### 2. Run the npm script outside the sandbox

Ask the Agent to run `npm run` / `npm run -w …` with Shell `required_permissions: ["all"]` from the start. A project rule such as `.cursor/rules/out-of-sandbox.mdc` can require that for every `package.json` script.

Outside the sandbox, CPU detection and Chromium launch work against the host cache. Outside the sandbox alone does **not** clear `PLAYWRIGHT_BROWSERS_PATH`; step 1 is still required.

## What did not work

- Deleting `process.env.PLAYWRIGHT_BROWSERS_PATH` in `playwright.config.ts` after `import '@playwright/test'`. ESM runs imports first, and the CLI has already fixed the registry directory.
- Only `env -u` while the command still runs inside the sandbox. Architecture detection and Chromium bootstrap still fail.
- Only running outside the sandbox without `env -u`. The empty `cursor-sandbox-cache` path remains and Playwright still re-downloads.
- Turning off the whole Agent sandbox (`Run Everything`, or `"type": "insecure_none"` in `sandbox.json`) only to fix Playwright. That is broader than needed.
- Adding the host Playwright cache to sandbox `additionalReadwritePaths`, or relying on `enableSharedBuildCache`. Cursor still redirects browsers via `PLAYWRIGHT_BROWSERS_PATH`.
- Expecting a Cursor setting that runs only Playwright outside the sandbox. No such per-command setting exists; use the project rule plus `env -u` instead.

## References

- Playwright browser downloads and `PLAYWRIGHT_BROWSERS_PATH`: [Browsers](https://playwright.dev/docs/browsers)
- Cursor project rule pattern: require `required_permissions: ["all"]` for `npm run` scripts (see `.cursor/rules/out-of-sandbox.mdc` in projects that already adopted this)
