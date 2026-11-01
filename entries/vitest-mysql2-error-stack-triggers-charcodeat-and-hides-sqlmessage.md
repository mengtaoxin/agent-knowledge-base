---
title: Vitest mysql2 error stack triggers charCodeAt and hides sqlMessage
tags: [vitest, vite, mysql2, drizzle-orm]
agents: [all]
symptoms:
  - TypeError: code.charCodeAt is not a function
  - Startup Error: code.charCodeAt is not a function during Vitest global setup
  - No test files found plus a charCodeAt crash with no ECONNREFUSED or sqlMessage
  - Real MySQL sqlMessage or ECONNREFUSED never appears in the Vitest failure output
  - Vitest process exits with Node.js version banner instead of a normal test failure summary
updated: 2026-11-01
---

# Vitest mysql2 error stack triggers charCodeAt and hides sqlMessage

## Problem

When Vitest runs API tests that talk to MySQL through `mysql2`, a real database failure (connection refused, illegal SQL, create-database failure) often never shows up as itself. The runner instead prints `TypeError: code.charCodeAt is not a function`, sometimes as a `Startup Error` with `No test files found`, and the Node process can exit before Vitest prints a normal failure summary.

Agents then misread the log as “MySQL is down” or as a Vitest/Vite bug, even when the underlying problem is bad SQL, a missed catch around `createDatabase`, or a connection error that never became a Promise rejection.

## Context

- Vitest with Vite’s module runner (Vitest’s default Node transform path)
- `mysql2` promise API / pool queries (also under Drizzle’s `Failed query` wrapper)
- Observed in a Node API test suite that creates scratch databases and runs migrations in `globalSetup`
- Versions in the originating project were recent Vitest 3.x / Vite and `mysql2` 3.x; the failure is from the stack-formatting interaction, not from one pin

Two failure shapes share the same surface:

1. **Promise rejects, print path blows up.** MySQL already rejected (for example illegal `DEFAULT` on `TEXT`). Vitest’s `printErrorInner` walks the `mysql2` error; Vite’s `prepareStackTrace` throws `charCodeAt`, and `sqlMessage` never appears.
2. **Stack rewrite throws before reject.** On some connection failures, `mysql2` reads `holder.stack` inside `applyCapturedStack` *before* `reject(err)`. Vite’s interceptor throws there, so the original `ECONNREFUSED` never becomes a rejected Promise and the process dies as an uncaught exception.

## Why

`mysql2` rewrites error stacks by reading `.stack` (in `applyCapturedStack`) so the frames point at the caller. Reading `.stack` invokes the global `Error.prepareStackTrace`. Vitest’s Vite module runner replaces that function to map runtime frames back to source. While doing so it calls `extractInlineSourceMap(code)` and assumes `code` is a string:

```js
code.charCodeAt(dataEnd - 1)
```

For some already-loaded modules, `meta.code` is not a string, so the interceptor throws `TypeError: code.charCodeAt is not a function`. That throw happens on the path that was supposed to surface the MySQL error. The real `sqlMessage`, `errno`, or `ECONNREFUSED` is lost; only the secondary `charCodeAt` remains.

Wrapping the error as a plain `Error` with an assigned string `stack` avoids the interceptor: assigning `error.stack = ...` skips Vite’s rewrite, so Vitest can print the message. Unwrapping Drizzle’s `cause` is required when the useful fields live under the wrapper, not on the top-level error.

Catching only `migrate()` is not enough. Any path that still hands a raw `mysql2` error to Vitest (for example `createDatabase` / `dropDatabase` outside the same helper, or a connection failure that throws inside `applyCapturedStack` before reject) can reproduce `charCodeAt`. Treating every `charCodeAt` as “MySQL unreachable” also fails: the same string appears for bad SQL and for print-path crashes when the server is fine.

## What worked

Rebuild MySQL failures into a plain `Error` before Vitest prints them:

1. Prefer `sqlMessage` (and SQL / errno / `sqlState`) from the error or its `cause`.
2. Include string connection codes such as `ECONNREFUSED` and `PROTOCOL_*`, not only numeric `errno`.
3. Assign a string `stack` on the new `Error` so Vite never runs `prepareStackTrace` on the mysql2 frames.
4. Apply that helper at every DB boundary that can fail during tests: migrate, create/drop/clone database, and `globalSetup` (log the formatted message, then rethrow).
5. Agent rules: stop for clear connection failures (`ECONNREFUSED`, timeouts, connection lost). Do not treat bare `charCodeAt` as proof the server is down; if `sqlMessage` / `ER_*` is present, fix the SQL.

Minimal shape:

```ts
function plainError(message: string): Error {
  const error = new Error(message);
  error.stack = `${error.name}: ${error.message}`;
  return error;
}

// formatMysqlError(err) -> plainError(`${sqlMessage} SQL: … (errno …)`)
```

## What did not work

- Reading `charCodeAt` as the root cause. It is the Vite stack rewriter failing while formatting a `mysql2` error.
- Wrapping only `migrate()` / migration catch paths. `createDatabase` and similar entry points still leaked raw `mysql2` errors into Vitest.
- A project rule that maps every `charCodeAt` to “MySQL unreachable”. That hid illegal SQL and leftover print-path bugs when the server was healthy.
- Turning off Vitest file parallelism. Serial files reduce load on MySQL; they do not stop the stack-formatting crash or restore `sqlMessage`.

## References

- `mysql2` `lib/promise/capture_local_err.js` (`applyCapturedStack`)
- Vite `dist/node/module-runner.js` (`extractInlineSourceMap` / `prepareStackTrace`)
