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
updated: 2026-11-05
---

# Cursor Agent re-downloads Playwright browsers every run

## Problem

When a Cursor Agent runs Playwright tests, each session downloads browser binaries again even though the host already has them under the normal Playwright cache. The download lands in a per-session directory that disappears when the chat ends, so the next run repeats the install.

Cursor injects `PLAYWRIGHT_BROWSERS_PATH` into the Agent shell. The value points at an empty path under `cursor-sandbox-cache`. Playwright treats that empty directory as the browser registry, so it reinstalls. Leaving the sandbox alone does not clear the variable: child processes still inherit it.

The shared `test:e2e` script also runs in CI and in other agents. Those runs do not have Cursor's sandbox path. An earlier fix cleared or rewrote `PLAYWRIGHT_BROWSERS_PATH` on that shared script, so every environment carried the workaround. The run that needs the host cache is a separate script, and it is started from the system terminal, outside the Agent sandbox.

## Context

- Cursor Desktop Agent with the default command sandbox
- Playwright / `@playwright/test` (any recent version that honors `PLAYWRIGHT_BROWSERS_PATH`)
- Observed on macOS. The host cache differs by OS: `~/Library/Caches/ms-playwright` on macOS, `~/.cache/ms-playwright` on Linux, `%LOCALAPPDATA%\ms-playwright` on Windows
- This behavior is Cursor-specific. Other coding agents that do not inject `PLAYWRIGHT_BROWSERS_PATH` or a Seatbelt sandbox do not need this entry
- In the projects that hit this, `npm run test` is the unit suite (Vitest). The shared Playwright script is `npm run test:e2e`. The host-cache script is `npm run test:e2e:local`

Two separate failures stack:

1. **Wrong registry directory.** Even with `required_permissions: ["all"]`, `PLAYWRIGHT_BROWSERS_PATH` still points at the empty sandbox cache path.
2. **Sandbox cannot run the host browsers.** Inside the sandbox, `os.cpus()` is empty, so Playwright may pick the wrong host platform (for example `mac-x64` on Apple Silicon). Chromium also fails to start (`bootstrap_check_in ... Permission denied`), and writing a fresh install into the host cache is blocked.

## Why

The re-download and the launch failure come from two independent injections, so clearing only one of them leaves the other intact.

Cursor sets `PLAYWRIGHT_BROWSERS_PATH` to an empty directory under `cursor-sandbox-cache`. Playwright reads that variable when `playwright-core` loads and treats the directory as its browser registry. The install therefore lands in a path that disappears when the chat ends, and the next run repeats it. Assigning the host cache in the process that starts Node, before any Playwright import, makes Playwright use the browsers already on the machine. Assigning it later, inside config that imports Playwright first, is too late: the registry path is already fixed.

The sandbox is a separate cause. With the host path set, Playwright looks at that cache, but inside the sandbox `os.cpus()` is empty, Chromium's bootstrap is denied, and writing a fresh install into the host cache is blocked. The script cannot leave the sandbox after it has started. The system terminal is outside that sandbox, so CPU detection and Chromium launch use the host cache. Setting the host path from inside the Agent shell still leaves the process in the sandbox, so Chromium does not start.

The shared `test:e2e` script stays `playwright test`. CI and any other runner keep calling it and keep whatever `PLAYWRIGHT_BROWSERS_PATH` they set. The host-cache assignment lives only on `test:e2e:local`. That script exports the platform default, then starts Playwright, so an inherited `cursor-sandbox-cache` value does not win.

Earlier versions of this entry put the workaround on the shared script, or asked the Agent to add it. An unconditional `env -u` deletes a real cache path along with Cursor's empty one. A `case` that unsets the variable only when the value contains `cursor-sandbox-cache` still changes the script every environment executes, and inside the sandbox it still cannot launch Chromium. A project rule that says "prefix the command" is only a reminder: the Agent can run the bare script. The public workaround that matches both causes is a separate script that points at the host cache, started from the system terminal. That is the form recorded here.

Broader workarounds miss one of the two causes. Turning off the Agent sandbox for every command is wider than this failure. Adding the host cache to sandbox write paths, or relying on a shared build cache, does not override `PLAYWRIGHT_BROWSERS_PATH`. `sandbox.json` with `enableSharedBuildCache: false` does not opt out of the injected path. There is no Cursor setting that runs only Playwright outside the sandbox.

## What worked

Use a separate script for the host cache, and run it from the system terminal. The shared script stays plain.

### 1. Point a local script at the host cache, before Playwright starts

`test:e2e` is the script CI and other environments run. `test:e2e:local` is the one that replaces Cursor's path.

```json
"test:e2e": "playwright test",
"test:e2e:local": "sh scripts/test-e2e-local.sh"
```

`scripts/test-e2e-local.sh` exports the platform cache, then replaces the shell with Playwright. Arguments after `npm run test:e2e:local --` are forwarded.

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

Playwright reads `PLAYWRIGHT_BROWSERS_PATH` when `playwright-core` loads. The export happens in the shell that starts it, so the child sees the host cache. `npm run test` (the unit suite) does not use this script.

### 2. Run that script from the system terminal

Run `npm run test:e2e:local` in the system terminal. That terminal is outside the Agent sandbox, so CPU detection and Chromium launch work against the host cache.

The Agent shell is the wrong place for this script. `required_permissions: ["all"]` disables the Seatbelt rules and still leaves `PLAYWRIGHT_BROWSERS_PATH` pointing at `cursor-sandbox-cache`. The local script replaces that path, and the system terminal is what gets the process out of the sandbox. Either half alone leaves one of the two failures in place.

## What did not work

- Putting `env -u PLAYWRIGHT_BROWSERS_PATH` on the shared `test:e2e` script. The unset then runs for CI and for a normal terminal. A runner that set `PLAYWRIGHT_BROWSERS_PATH` to a real cache loses it.
- A `case` on the shared `test:e2e` script that unsets the variable when the value contains `cursor-sandbox-cache`. Every environment then executes the Cursor check. Inside the sandbox, Chromium still fails to start.
- Telling the Agent to run `env -u PLAYWRIGHT_BROWSERS_PATH npm run test:e2e` and expecting it to add that prefix. That line is an instruction. The Agent does not test the variable, and a rule that says to run scripts as-is sends it back to the bare command.
- Deleting `process.env.PLAYWRIGHT_BROWSERS_PATH` in `playwright.config.ts` after `import '@playwright/test'`. ESM runs imports first, and the CLI has already fixed the registry directory.
- Setting the host cache path, or unsetting the variable, while the command still runs inside the sandbox. Architecture detection and Chromium bootstrap still fail.
- Only running outside the sandbox without replacing `PLAYWRIGHT_BROWSERS_PATH`. The empty `cursor-sandbox-cache` path remains and Playwright still re-downloads. `required_permissions: ["all"]` does not clear the variable.
- Turning off the whole Agent sandbox (`Run Everything`, or `"type": "insecure_none"` in `sandbox.json`) only to fix Playwright. That is broader than needed.
- Adding the host Playwright cache to sandbox `additionalReadwritePaths`, relying on `enableSharedBuildCache`, or setting `enableSharedBuildCache: false`. Cursor still redirects browsers via `PLAYWRIGHT_BROWSERS_PATH`.
- Expecting a Cursor setting that runs only Playwright outside the sandbox. No such per-command setting exists.

## References

- Playwright browser downloads and `PLAYWRIGHT_BROWSERS_PATH`: [Browsers](https://playwright.dev/docs/browsers)
- Playwright reads the variable when the module loads, so set it before the process starts: [microsoft/playwright#33819](https://github.com/microsoft/playwright/issues/33819)
- Cursor staff workaround: override `PLAYWRIGHT_BROWSERS_PATH` to the host cache before the test command. `enableSharedBuildCache: false` does not opt out: [Agent sandbox redirects PLAYWRIGHT_BROWSERS_PATH](https://forum.cursor.com/t/agent-sandbox-redirects-playwright-browsers-path-and-npm-config-devdir-to-empty-cursor-sandbox-cache-windows-playwright-reinstall-loops-npm-11-warnings/167542)
- Separate `test:e2e:local` script, run from the system terminal: [asap-protocol playwright-e2e.md](https://github.com/adriannoes/asap-protocol/blob/main/apps/web/docs/playwright-e2e.md)
