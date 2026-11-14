---
title: Run agent commands on a remote host, not a local sandbox
tags: [agents, sandbox, remote-host, vm, networking, testing]
agents: [all]
symptoms:
  - Agent keeps rewriting shell commands when the real failure is sandbox or firewall interception
  - Network requests from the agent fail or never arrive, and the agent treats that as a bad command
  - Isolating only unit tests still leaves other agent actions on the sandboxed local host
  - The whole coding loop keeps detouring around local sandbox and firewall limits
updated: 2026-11-15
---

# Run agent commands on a remote host, not a local sandbox

## Problem

Coding agents often execute on the developer's laptop under a local sandbox or firewall. When a command or network request is blocked, the payload may never reach the intended process or host. The agent does not reliably see “blocked by sandbox.” It sees a failed command, a timeout, or a missing response, and then treats the *command* as the bug. It rewrites flags, changes URLs, invents workarounds, and burns the session on the wrong cause.

Putting only the unit-test runner in an isolated box is not enough. Tests may dodge some host limits, but other agent actions — especially network requests — still depend on the sandboxed local shell. The detour then spreads from “how we run tests” to the entire agent interaction loop.

Dev Containers were considered as a way to move the whole workspace into a container on the laptop. They still leave day-to-day agent execution in the local product surface the author finds awkward, and they are the wrong shape for “get out of the local sandbox entirely.”

## Context

- Coding agents that run shell commands and network calls from a local machine (including Cursor Agent under its sandbox)
- A self-built or rented virtual machine, or a physical host, used as a remote development environment
- The project workspace lives on that remote host; the laptop is an editor or entry point, not the place where agent commands execute
- Scope is the full agent runtime (tests, servers, package installs, outbound network), not a single test command
- Not Dev Containers, and not “keep editing and running on the laptop while only tests opt into Docker”

## Why

### Why the failure happens

A local agent sandbox or firewall can drop or rewrite commands and network traffic. The observable symptom is usually a failed invocation, not an explicit “permission denied by policy” the agent can trust. Once the agent assumes the command text is wrong, it optimizes the wrong variable. Every later step in the chat inherits that misdiagnosis.

Unit-test isolation does not close this hole. Any agent turn that still shells out or fetches from the laptop stays on the sandboxed path. The interaction itself becomes the detour: humans and agents spend attention on command shape while the environment is what is broken.

### What running on a remote host preserves

- **Honest failure signals.** Commands and network calls either work or fail for reasons visible in that environment, instead of vanishing under a local policy layer the agent cannot see clearly.
- **One place for the whole agent runtime.** Tests, servers, installs, and outbound requests share the same unconstrained (or at least non-laptop-sandbox) host, so the suite is not a special case while everything else still fights the sandbox.
- **A laptop that stays thin.** The machine in front of the human is for editing and connecting. The remote host holds the workspace and runs the work the agent needs done.

### Why this approach, not the alternatives

- **Keep agent execution on the laptop and keep fighting the sandbox.** Each blocked call invites another command rewrite. The root cause is the execution venue, not the argv.
- **Isolate only automated tests (for example run the suite in Docker on the laptop).** Narrower than the failure. Network and other non-test agent actions still hit the sandbox and still mislead the agent.
- **Dev Containers for the whole local workspace.** Moves tools into a container but keeps the loop tied to the local Dev Container workflow. The author finds that model awkward and still not the same as “run the agent’s work on a host that is not under the laptop agent sandbox.”
- **Always assume the command is wrong when something fails.** That habit is exactly how sandbox interception turns into thrashing.

### Author's path and judgment

An earlier entry argued for running automated tests in Docker on the local machine, with ordinary edit and lint left on the host. That entry was deleted on purpose. Following the sandbox pain further showed the real cost was not “Playwright or unit tests on bare metal.” It was the agent spending turns repairing commands that never had a fair chance to run. Once network and other requests kept getting intercepted, the whole chat became a bypass exercise. The cleaner rule is to move the workspace and agent command execution onto a remote VM or physical host, and treat the laptop as the editor side only. Dev Containers were rejected again: the goal is an ordinary remote environment the author controls, not a local container-bound IDE workflow.

## What worked

1. Provision a remote host: a new virtual machine or a real machine you control.
2. Put the project workspace on that host. Sync or clone so the code the agent runs is the code on the remote machine.
3. Run the agent’s command surface there — installs, tests, servers, and network calls — in an environment that does not sit behind the laptop’s agent sandbox.
4. Use the local machine as the editor or entry point only. Do not keep the primary agent execution loop on the sandboxed laptop shell.
5. When something fails, debug the remote environment and the real command outcome. Do not start by assuming a sandbox-blocked call means the command text needs another rewrite.

Shape only (exact SSH, sync, or editor remote wiring is project-specific):

```text
Laptop: editor / connection
Remote host: workspace + agent command execution (tests, network, services)
```

## What did not work

- Leaving agent execution on the local sandboxed host and treating blocked or dropped network calls as bad commands to keep editing.
- Isolating only the unit-test path while the rest of the agent loop stayed on the laptop under sandbox and firewall rules.
- Adopting Dev Containers so the whole local workspace runs in a container. The author still did not want that day-to-day model; it did not replace the need for a remote host outside the laptop sandbox fight.

## References

- Related suite-selection policy once tests already run somewhere stable: [prefer-full-unit-suite-over-selective-test-routing.md](prefer-full-unit-suite-over-selective-test-routing.md)
