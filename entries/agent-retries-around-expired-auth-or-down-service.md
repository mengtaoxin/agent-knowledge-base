---
title: Agent retries around expired auth or a down service instead of stopping
tags: [agent-behavior, auth, credentials, mcp, external-services]
agents: [all]
symptoms:
  - GitHub token or other credential is expired; MCP, CLI, and API each fail with 401 or auth errors
  - Agent switches between MCP, plugins, and CLI to work around the same auth failure
  - Local MySQL or another required service is down (ECONNREFUSED, connection refused); agent keeps inventing bypasses
  - User is never asked to renew the credential or start the service
updated: 2026-11-02
---

# Agent retries around expired auth or a down service instead of stopping

## Problem

A coding agent hits a blocker that only the user can fix: an expired or missing credential, or a required local or remote service that is not running. Instead of stopping, the agent treats the failure as a bad choice of tool. It retries the same capability through another surface — API, then MCP, then a plugin or CLI — or invents workarounds for a database that is simply down. Each attempt fails for the same root cause. The user sees busy exploration and never gets a clear request to renew the token or start the service.

Autonomous search for another approach is still the right default when the failure is a wrong path, a wrong API shape, or an agent mistake. The failure mode here is continuing that search after the failure class has already been identified as an external precondition the agent cannot restore.

## Context

Any coding agent that can call several tools for one capability (for example GitHub via MCP, the GitHub CLI, and HTTP APIs), or that depends on services outside the conversation (local MySQL, Redis, a remote API). The lesson is about when to stop and hand control back, not about how to refresh a specific product's token.

## Why

### Why the retries do not help

The root cause is not the tool surface. An expired GitHub token fails every client that uses that token. A stopped MySQL instance refuses every client library. Switching MCP → CLI → plugin does not change the authority or the process that is missing. The new attempt only burns turns and context; it does not produce new diagnostic information once the class of failure is known.

### What stopping preserves

Stopping after the class is clear keeps three things intact:

- The user sees the real blocker (renew the credential, start the service) instead of a trail of equivalent failures.
- The agent does not quietly bypass the intended control plane (for example scraping a site because the authenticated API is 401) without consent.
- Work that does not depend on the blocker can still proceed; only the blocked segment pauses.

### Why this approach

The decision is by **failure class**, not by “are there more tools left?”

- **Keep trying** when the failure is local to an approach: wrong path, wrong assumption, ambiguous or transient error, or another implementation that still has the same authority. One more attempt can learn something new.
- **Stop and confirm** when the failure is an external authority or environment precondition the agent cannot restore: expired or missing credentials, permissions only a human can grant, or a required service the user must start. Confirm with a short choice: wait for the user to fix it and retry, or (only if they allow it) search for a bypass.

A project-local rule that only lists “stop on ECONNREFUSED” or “stop on 401” helps for those strings. It does not replace the class test: the same strings can appear once as noise; after the class is confirmed across attempts, further tool-hopping is the bug.

Related: connection failures also appear under Vitest and `mysql2`, where a secondary print error can hide `ECONNREFUSED`. See [vitest-mysql2-error-stack-triggers-charcodeat-and-hides-sqlmessage.md](./vitest-mysql2-error-stack-triggers-charcodeat-and-hides-sqlmessage.md). That entry is about recovering the real SQL or connection error; this entry is about stopping once that real error is known to be an unrestorable precondition.

## What worked

When several channels fail with the same auth or connectivity class, or a single clear signal already names that class:

1. Name the blocker in plain language (expired token, service not running).
2. Stop further equivalent retries.
3. Ask the user which path to take, for example: wait until they renew the credential or start the service, then retry; or allow a stated bypass if one exists and they want it.
4. Continue only the parts of the task that do not need that blocker.

Example confirmation shape:

> The GitHub token appears expired (401 across CLI and MCP). Should I wait for you to renew it and then retry, or do you want me to look for a path that does not use that token?

## What did not work

- Treating every tool failure as a cue to try another integration for the same capability without classifying the error.
- Inventing application-level bypasses for a down local database instead of asking the user to start it.
- Continuing to explore after two or more surfaces already reported the same 401 or connection-refused root cause.

## References

- Related entry on MySQL connection failures hidden by Vitest stack formatting: [vitest-mysql2-error-stack-triggers-charcodeat-and-hides-sqlmessage.md](./vitest-mysql2-error-stack-triggers-charcodeat-and-hides-sqlmessage.md)
