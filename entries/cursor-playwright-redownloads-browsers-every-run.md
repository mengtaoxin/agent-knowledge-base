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
updated: 2026-11-07
---

# Cursor Agent re-downloads Playwright browsers every run

## Problem

When a Cursor Agent runs Playwright tests, each session downloads browser binaries again even though the host already has them under the normal Playwright cache. The download lands in a per-session directory that disappears when the chat ends, so the next run repeats the install.

Cursor injects `PLAYWRIGHT_BROWSERS_PATH` into the Agent shell. The value points at an empty path under `cursor-sandbox-cache`. Playwright treats that empty directory as the browser registry, so it reinstalls. Leaving the sandbox alone does not clear the variable: child processes still inherit it, including when the Shell tool uses `required_permissions: ["all"]`.

Two failures can stack:

1. **Wrong registry directory.** `PLAYWRIGHT_BROWSERS_PATH` still points at the empty sandbox cache path.
2. **Sandbox cannot run the host browsers.** Inside the Seatbelt sandbox, `os.cpus()` may be empty (wrong host platform, for example `mac-x64` on Apple Silicon), Chromium fails to start (`bootstrap_check_in ... Permission denied`), and writing a fresh install into the host cache is blocked. `required_permissions: ["all"]` turns Seatbelt off for that command; it does not clear the variable.

## Context

- Cursor Desktop Agent with the default command sandbox
- Playwright / `@playwright/test` (any recent version that honors `PLAYWRIGHT_BROWSERS_PATH`)
- Observed on macOS. The host cache differs by OS: `~/Library/Caches/ms-playwright` on macOS, `~/.cache/ms-playwright` on Linux, `%LOCALAPPDATA%\ms-playwright` on Windows
- Cursor-specific. Other coding agents that do not inject `PLAYWRIGHT_BROWSERS_PATH` or a Seatbelt sandbox do not need this entry
- In the projects that hit this, `npm run test` is the unit suite (Vitest). The shared Playwright script is `npm run test:e2e`. The host-cache script is `npm run test:e2e:local`. A finish gate such as `npm run verify -- --e2e` may call one of those scripts

## Why

### Why the failure happens

The re-download and the launch failure come from two independent injections. Clearing only one leaves the other intact.

Cursor sets `PLAYWRIGHT_BROWSERS_PATH` to an empty directory under `cursor-sandbox-cache`. Playwright reads that variable when `playwright-core` loads and treats the directory as its browser registry. The install lands in a path that disappears when the chat ends. Assigning the host cache in the process that starts Node, before any Playwright import, makes Playwright use the browsers already on the machine. Assigning it later, inside config that imports Playwright first, is too late: the registry path is already fixed.

Seatbelt is separate. With the host path set but still inside the sandbox, architecture detection and Chromium bootstrap can still fail. `required_permissions: ["all"]` disables Seatbelt for that command and still leaves the injected path. Replacing the path (via `test:e2e:local` or an equivalent export) plus running outside Seatbelt — either with `["all"]` in the Agent shell, or in the system terminal — clears both causes.

### What the fix preserves

- Shared `test:e2e` stays `playwright test`. CI and other runners keep calling it and keep whatever `PLAYWRIGHT_BROWSERS_PATH` they set.
- Host-cache assignment lives only on `test:e2e:local` (or an equivalent local wrapper). An inherited `cursor-sandbox-cache` value does not win once that script exports the platform default.
- A finish gate that would otherwise call shared `test:e2e` can detect the Cursor sandbox path and run `test:e2e:local` instead, so Agent `verify --e2e` does not re-download without teaching every caller to remember the local script name.

### Why this approach

Putting the workaround on shared `test:e2e` forces every environment to carry a Cursor check. An unconditional `env -u` deletes a real cache path along with Cursor's empty one. A project rule that says "prefix the command" is only a reminder: the Agent can run the bare script.

Requiring the system terminal alone is stronger than needed once Seatbelt is off and the path is rewritten. The durable pieces are: a separate local script that sets the host cache before Playwright starts, permission to leave Seatbelt when the Agent runs that script, and optionally a finish gate that picks the local script when `PLAYWRIGHT_BROWSERS_PATH` contains `cursor-sandbox-cache`.

Earlier versions of this entry put the workaround on the shared script, asked the Agent to add a prefix, or treated the system terminal as the only place that could launch Chromium. The form recorded here matches both causes without changing CI's shared script.

Broader workarounds miss one of the two causes. Turning off the Agent sandbox for every command is wider than this failure. Adding the host cache to sandbox write paths, or relying on a shared build cache, does not override `PLAYWRIGHT_BROWSERS_PATH`. `sandbox.json` with `enableSharedBuildCache: false` does not opt out of the injected path. There is no Cursor setting that runs only Playwright outside the sandbox.

## What worked

Keep a separate host-cache script. Optionally teach the finish gate to pick it under Cursor. Run with Seatbelt off when the Agent owns the command.

### 1. Point a local script at the host cache, before Playwright starts

`test:e2e` is what CI and other environments run. `test:e2e:local` replaces Cursor's path.

```json
"test:e2e": "playwright test",
"test:e2e:local": "sh scripts/test-e2e-local.sh"
```

```sh
#!/bin/sh
case "$(uname -s)" in
  Darwin) PLAYWRIGHT_BROWSERS_PATH="$HOME/Library/Caches/ms-playwright" ;;
  Linux) PLAYWRIGHT_BROWSERS_PATH="$HOME/.cache/ms-playwright" ;;
  MINGW* | MSYS* | CYGWIN*) PLAYWRIGHT_BROWSERS_PATH="${LOCALAPPDATA}/ms-playwright" ;;
  *) echo "unsupported OS: $(uname -s)" >&2; exit 1 ;;
esac
export PLAYWRIGHT_BROWSERS_PATH
exec playwright test "$@"
```

Playwright reads `PLAYWRIGHT_BROWSERS_PATH` when `playwright-core` loads. The export happens in the shell that starts it. Arguments after `npm run test:e2e:local --` are forwarded. The unit suite does not use this script.

### 2. Run that script outside Seatbelt

In the Agent shell, run `npm run test:e2e:local` with `required_permissions: ["all"]` so Seatbelt is off and the local script has already replaced the sandbox path. The system terminal also works: it is outside the Agent sandbox by default. Path rewrite alone inside Seatbelt is not enough; Seatbelt off alone without rewriting the path still re-downloads into `cursor-sandbox-cache`.

### 3. Finish gates: auto-pick the local script under Cursor

When a command such as `npm run verify -- --e2e` would otherwise call shared `test:e2e`, detect Cursor's empty registry and call `test:e2e:local` instead:

```ts
function e2eNpmScript(browsersPath?: string): 'test:e2e' | 'test:e2e:local' {
  const path = arguments.length === 0 ? process.env.PLAYWRIGHT_BROWSERS_PATH : browsersPath;
  if (path?.includes('cursor-sandbox-cache')) return 'test:e2e:local';
  return 'test:e2e';
}
```

CI and a normal terminal keep `test:e2e`. The Agent finish gate stops re-downloading without changing the shared script.

## What did not work

- Putting `env -u PLAYWRIGHT_BROWSERS_PATH` on the shared `test:e2e` script. The unset then runs for CI and for a normal terminal. A runner that set `PLAYWRIGHT_BROWSERS_PATH` to a real cache loses it.
- A `case` on the shared `test:e2e` script that unsets the variable when the value contains `cursor-sandbox-cache`. Every environment then executes the Cursor check. Inside Seatbelt, Chromium still fails to start.
- Telling the Agent to run `env -u PLAYWRIGHT_BROWSERS_PATH npm run test:e2e` and expecting it to add that prefix. That line is an instruction. The Agent does not test the variable, and a rule that says to run scripts as-is sends it back to the bare command.
- Deleting `process.env.PLAYWRIGHT_BROWSERS_PATH` in `playwright.config.ts` after `import '@playwright/test'`. ESM runs imports first, and the CLI has already fixed the registry directory.
- Setting the host cache path, or unsetting the variable, while the command still runs inside Seatbelt. Architecture detection and Chromium bootstrap still fail.
- Only running outside Seatbelt without replacing `PLAYWRIGHT_BROWSERS_PATH`. The empty `cursor-sandbox-cache` path remains and Playwright still re-downloads. `required_permissions: ["all"]` does not clear the variable.
- Turning off the whole Agent sandbox (`Run Everything`, or `"type": "insecure_none"` in `sandbox.json`) only to fix Playwright. That is broader than needed.
- Adding the host Playwright cache to sandbox `additionalReadwritePaths`, relying on `enableSharedBuildCache`, or setting `enableSharedBuildCache: false`. Cursor still redirects browsers via `PLAYWRIGHT_BROWSERS_PATH`.
- Expecting a Cursor setting that runs only Playwright outside the sandbox. No such per-command setting exists.

## References

- Playwright browser downloads and `PLAYWRIGHT_BROWSERS_PATH`: [Browsers](https://playwright.dev/docs/browsers)
- Playwright reads the variable when the module loads, so set it before the process starts: [microsoft/playwright#33819](https://github.com/microsoft/playwright/issues/33819)
- Cursor staff workaround: override `PLAYWRIGHT_BROWSERS_PATH` to the host cache before the test command. `enableSharedBuildCache: false` does not opt out: [Agent sandbox redirects PLAYWRIGHT_BROWSERS_PATH](https://forum.cursor.com/t/agent-sandbox-redirects-playwright-browsers-path-and-npm-config-devdir-to-empty-cursor-sandbox-cache-windows-playwright-reinstall-loops-npm-11-warnings/167542)
- Separate `test:e2e:local` script (host cache before Playwright starts): [asap-protocol playwright-e2e.md](https://github.com/adriannoes/asap-protocol/blob/main/apps/web/docs/playwright-e2e.md)
