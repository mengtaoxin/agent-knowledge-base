---
title: Agent adds defensive compatibility to silence contract errors
tags: [agent-behavior, defensive-coding, api-contracts, errors, maintainability]
agents: [all]
symptoms:
  - A method rejects a wrong-type or out-of-contract argument (for example Object expected as number 1, caller passes string "1"); the agent converts or dual-accepts the value so the call no longer throws
  - Compatibility or coercion is added inside core business logic to minimize reported errors
  - Environment failures (database unreachable, dependency down) are treated as defects to paper over instead of signals to surface
  - Defensive branches accumulate in large libraries and raise line count, bug surface, and review cost
updated: 2026-11-02
---

# Agent adds defensive compatibility to silence contract errors

## Problem

When a call fails because the argument or environment violates an expected contract, coding agents (and many human reviewers) default to making the failure go away. Typical moves: coerce `"1"` to `1`, accept both shapes in the same method, catch and continue, or invent a soft fallback. The immediate error count drops, so the change looks like a fix.

That habit is wrong when the throw was the useful signal. Not every failure means the callee's logic is broken. A database that will not connect should fail loudly. A parameter that is outside the documented contract should stay a hard error until the interface is deliberately redesigned. Blind compatibility grows line count and maintenance cost; in large, high-quality libraries that cost compounds into real defect rate.

## Context

General coding-agent behavior across languages. The motivating illustration is a Java-style API that takes `Object`, documents that only the number `1` runs a special path, and throws for every other value — including the string `"1"`. The same pattern appears whenever an agent meets a type, enum, status, or precondition mismatch and reaches for coercion instead of preserving the contract.

Related but separate: when the failure is an unrestorable external precondition (expired auth, service down), agents also tend to keep retrying. See [agent-retries-around-expired-auth-or-down-service.md](./agent-retries-around-expired-auth-or-down-service.md). That entry is about stopping tool-hopping; this entry is about refusing to paper over contract and environment errors with defensive code.

## Why

### Why the silence looks like a fix

Training and day-to-day coding pressure both reward “make it run.” An exception in a log or a red test is treated as something to eliminate. Coercion and dual acceptance are cheap locally and often pass the current call site, so they are the path of least resistance for an agent mid-edit.

### Why that is often the wrong mechanism

- **Environment errors are information.** A refused database connection reports that the world is wrong, not that the connection code must swallow the failure. Surfacing it is correct behavior.
- **Contract errors are design feedback.** If callers pass `"1"` when only `1` is valid, the useful responses are: keep throwing, or change the public contract on purpose. Quietly accepting both embeds ambiguity into every future caller and every future model edit.
- **Line count is a reliability signal.** Extra defensive branches are not free. More paths mean more bugs, more review surface, and harder long-term maintenance for humans who still have to read the code. High-standard or very large libraries feel this first.
- **Over-defense and core logic should not merge.** Defensive code is sometimes necessary, but burying it inside the main algorithm makes both the algorithm and the defense harder to reason about.

### Why this approach

Alternatives that fail the cause:

- **Always minimize errors** — treats every throw as a defect. Collapses good failures (env and contract) with bad ones (true logic bugs).
- **Compat inside the hot path “just this once”** — grows into permanent dual semantics without an explicit API decision.
- **Leave the decision to the next model turn** — models are biased toward more defense and toward covering errors; without a maintained rule, the next edit repeats the silence.

Prefer: keep honest failures; change the contract only deliberately; if defense is required, isolate it (for example a dedicated parameter-check helper) so core logic stays readable.

## What worked

When an agent or a patch wants to “fix” a throw that comes from a wrong argument shape or a bad environment:

1. Classify the failure before editing: true logic bug, broken environment, or caller/contract mismatch.
2. For environment and contract mismatches, prefer leaving the error thrown (or making it clearer) over coercion that only serves the current call site.
3. Do not add string/number (or similar) dual acceptance inside core business methods unless the public contract is being redesigned on purpose and documented that way.
4. If defensive checks are truly needed, put them in a named check/validation layer in front of the algorithm, not mixed into the main path.
5. When unsure whether to compat or throw, throw (or ask). Silence is the choice that is hardest to reverse in a large codebase.

Illustration of the pressure to resist (documentation only):

```java
// Documented: only Integer 1 takes the special path; anything else throws.
void handle(Object input) {
    if (Integer.valueOf(1).equals(input)) {
        runSpecialPath();
        return;
    }
    throw new IllegalArgumentException("expected Integer 1, got: " + input);
}

// Resist the “helpful” patch that only hides the bad call site:
// if ("1".equals(input) || Integer.valueOf(1).equals(input)) { ... }
```

If validation must grow, prefer a separate check rather than expanding the core method:

```java
void checkHandleInput(Object input) { /* explicit defense lives here */ }

void handle(Object input) {
    checkHandleInput(input);
    runSpecialPath(); // core path stays narrow
}
```

## What did not work

- Treating “fewer thrown errors” as the primary quality goal for library code.
- Coercing `"1"` to `1` (or accepting both) inside the business method so a mistaken call succeeds.
- Stacking more defensive branches into the same method that owns the core algorithm.
- Assuming every agent- or human-proposed compatibility shim is low-cost because it is only a few lines.

## References

- Related entry on stopping when auth expires or a required service is down: [agent-retries-around-expired-auth-or-down-service.md](./agent-retries-around-expired-auth-or-down-service.md)
