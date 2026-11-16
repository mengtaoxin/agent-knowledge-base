---
title: Agent retries around expired auth, down services, or unreachable environments instead of stopping
tags: [agent-behavior, auth, credentials, mcp, external-services, networking, environment]
agents: [all]
symptoms:
  - GitHub token or other credential is expired; MCP, CLI, and API each fail with 401 or auth errors
  - Agent switches between MCP, plugins, and CLI to work around the same auth failure
  - Agent opens a browser, reads .env, or inspects shell environment variables to bypass an expired token
  - Local MySQL or another required service is down (ECONNREFUSED, connection refused); agent keeps inventing bypasses
  - A URL or host is unreachable after a network or environment change; agent keeps probing, rewriting, or adding temporary check code
  - User is never asked to renew the credential, start the service, or confirm the environment
updated: 2026-11-16
---

# Agent retries around expired auth, down services, or unreachable environments instead of stopping

## Problem

A coding agent hits a blocker that only the user can fix: an expired or missing credential, a required local or remote service that is not running, or a host that is unreachable in the current network or environment. Instead of stopping, the agent treats the failure as a bad choice of tool or a bug in the call site. It retries the same capability through another surface — API, then MCP, then a plugin or CLI — opens a browser to click through what should be an authenticated API, reads `.env` or shell environment variables looking for a load mistake, invents workarounds for a database that is simply down, or writes a ring of temporary probe code against a URL that will not answer from this network. Each attempt fails for the same root cause. The user sees busy exploration and never gets a clear request to renew the token, start the service, or confirm that the environment itself changed.

Two classic shapes:

1. **Credential expiry.** A GitHub or other platform token is expired. A human regenerates the token and puts it back. The agent cannot do that, so it drifts into browser automation, secret-file inspection, and environment-loading checks that leave the real ask unstated.
2. **Network or environment shift.** Code written against an intranet or home-lab address is run from another place. The host is not wrong in the repository; the current network cannot reach it. The agent keeps checking and may spend a full loop of temporary diagnostics before it finally stops.

Autonomous search for another approach is still the right default when the failure is a wrong path, a wrong API shape, or an agent mistake. The failure mode here is continuing that search after the failure class has already been identified as an external precondition the agent cannot restore.

## Context

Any coding agent that can call several tools for one capability (for example GitHub via MCP, the GitHub CLI, and HTTP APIs), that can open a browser or read local config, or that depends on services and hosts outside the conversation (local MySQL, Redis, a remote API, an intranet URL). The lesson is about when to stop and hand control back, not about how to refresh a specific product's token or how to reconfigure a specific VPN.

The soft form of the rule matters as much as the stop itself. A project rule that forces confirmation on every uncertain step makes the agent pause on work it should just finish. The useful rule names a narrow class of unrestorable blockers and leaves ordinary debugging alone.

## Why

### Why the retries do not help

The root cause is not the tool surface. An expired GitHub token fails every client that uses that token. A stopped MySQL instance refuses every client library. An intranet host that is off the current network refuses every probe the agent invents. Switching MCP → CLI → plugin, opening a browser, rereading `.env`, or adding temporary connectivity scripts does not change the authority, the process, or the network path that is missing. The new attempt only burns turns and context; it does not produce new diagnostic information once the class of failure is known.

### What stopping preserves

Stopping after the class is clear keeps three things intact:

- The user sees the real blocker (renew the credential, start the service, fix the network or choose another host) instead of a trail of equivalent failures.
- The agent does not quietly bypass the intended control plane (for example scraping a site because the authenticated API is 401, or rewriting addresses without consent) without asking.
- Work that does not depend on the blocker can still proceed; only the blocked segment pauses.

### Why this approach

The decision is by **failure class**, not by “are there more tools left?”

- **Keep trying** when the failure is local to an approach: wrong path, wrong assumption, ambiguous or transient error, or another implementation that still has the same authority. One more attempt can learn something new.
- **Stop and confirm** when the failure is an external authority or environment precondition the agent cannot restore: expired or missing credentials, permissions only a human can grant, a required service the user must start, or a host that is unreachable because the current network or environment differs from the one the code assumed. Confirm with a short choice: wait for the user to fix it and retry, or (only if they allow it) search for a bypass.

Do not phrase the project rule as a hard stop on every failed request. Absolute wording pushes the agent to ask before ordinary edits and investigations, which costs more than it saves. Prefer class-based language: after the blocker is identified as unrestorable by the agent, stop equivalent retries and ask once; do not require a checkpoint for failures that still look like local mistakes.

A project-local rule that only lists “stop on ECONNREFUSED” or “stop on 401” helps for those strings. It does not replace the class test: the same strings can appear once as noise; after the class is confirmed across attempts, further tool-hopping is the bug. The same caution applies to network failures: one timeout is not enough to declare an environment shift; repeated failures that share the same unreachable host or the same auth class are.

Related: connection failures also appear under Vitest and `mysql2`, where a secondary print error can hide `ECONNREFUSED`. See [vitest-mysql2-error-stack-triggers-charcodeat-and-hides-sqlmessage.md](./vitest-mysql2-error-stack-triggers-charcodeat-and-hides-sqlmessage.md). That entry is about recovering the real SQL or connection error; this entry is about stopping once that real error is known to be an unrestorable precondition.

Also related: when the failure is a contract or type mismatch, agents often paper it over with defensive compatibility instead of keeping the throw. See [agent-adds-defensive-compat-to-silence-contract-errors.md](./agent-adds-defensive-compat-to-silence-contract-errors.md).

### Author's judgment and the soft boundary

The default agent pattern is “a problem arrived, so solve it.” That pattern is usually right. The pain shows up when the fix lives outside the agent's authority: a human must mint a token, start a process, or move onto the right network. In those cases the costly behavior is not the final stop; it is the unnecessary circle before the stop — browser detours, env-var archaeology, temporary probe scripts — after the class was already clear.

The suggested project rule is therefore not “always ask the user.” It is “when this class is clear, do not hard-solve; pause and confirm, then continue only what they allow.” Wording that is too absolute trains the opposite failure: the agent interrupts every ordinary task for permission. Soft, class-scoped wording keeps the default autonomy for fixable work and reserves the handoff for blockers the agent cannot restore.

## What worked

When several channels fail with the same auth, connectivity, or environment class, or a single clear signal already names that class:

1. Name the blocker in plain language (expired token, service not running, host unreachable from this network).
2. Stop further equivalent retries, including browser workarounds, `.env` / environment-variable fishing for the same credential, and temporary probe or rewrite code aimed at the same unreachable host.
3. Ask the user which path to take, for example: wait until they renew the credential, start the service, or fix the network, then retry; or allow a stated bypass if one exists and they want it.
4. Continue only the parts of the task that do not need that blocker.

Example confirmation shapes:

> The GitHub token appears expired (401 across CLI and MCP). Should I wait for you to renew it and then retry, or do you want me to look for a path that does not use that token?

> This host looks unreachable from the current network (repeated timeouts / connection failures). Should I wait while you switch network or VPN, point me at a reachable URL, or stop this part of the task?

Example soft project-rule shape (adapt; do not harden into “ask before every step”):

> If the failure is an unrestorable external precondition — expired or missing credentials, a service only the user can start, or a host unreachable because the environment or network changed — stop equivalent retries and ask once. Do not open browsers, dig through env files, or write temporary probe code for that same blocker unless the user asks. Keep normal autonomous debugging for ordinary code and path mistakes.

## What did not work

- Treating every tool failure as a cue to try another integration for the same capability without classifying the error.
- Opening a browser, reading `.env`, or auditing environment-variable loading to work around an expired token the user must renew.
- Inventing application-level bypasses for a down local database instead of asking the user to start it.
- Writing temporary connectivity checks or address rewrites in a loop when the host is simply off the current network.
- Continuing to explore after two or more surfaces already reported the same 401, connection-refused, or unreachable-host root cause.
- Writing the stop rule so absolutely that the agent starts confirming ordinary, agent-fixable work.

## References

- Related entry on MySQL connection failures hidden by Vitest stack formatting: [vitest-mysql2-error-stack-triggers-charcodeat-and-hides-sqlmessage.md](./vitest-mysql2-error-stack-triggers-charcodeat-and-hides-sqlmessage.md)
