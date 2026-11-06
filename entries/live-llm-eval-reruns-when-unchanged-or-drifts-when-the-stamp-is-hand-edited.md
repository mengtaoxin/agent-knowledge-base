---
title: Live LLM eval reruns when unchanged, or drifts when the stamp is hand-edited
tags: [llm, eval, fingerprint, cost, drift]
agents: [all]
symptoms:
  - The ordinary unit suite calls a live model and spends tokens when the prompt, cases, and model id have not changed
  - A prompt, model id, case file, or input builder changes and the ledger still says the call is tested
  - A hand-edited fingerprint is committed as if the live eval had passed
  - A machine with no model API key cannot commit because the unit suite requires a live eval
  - One live eval run spends tokens on call sites whose fingerprints already match
updated: 2026-11-07
---

# Live LLM eval reruns when unchanged, or drifts when the stamp is hand-edited

## Problem

A project that calls a paid model needs live evals. Those runs cost money, and they are slow. Putting them inside the ordinary unit suite spends that money on every commit, including commits that did not change anything the model sees.

Leaving them out entirely has the opposite failure. A prompt, a resolved model id, a case, or the code that builds the model input can change, and nothing forces a new eval. Quality drifts, and the drift is quiet.

A boolean "tested" flag does not fix the drift. It is written by hand, it stays true after the logic changes, and anyone can flip it back to true without running the model.

## Context

This applies to any repository with one or more paid model call sites: a prompt plus the code that turns inputs into a model request, scored against fixed cases. Deterministic logic stays in the unit suite and never opens a network connection. Live evals are separate commands, run by hand when a fingerprint says they are due.

The ledger is a checked-in file. Each record is for one call site, not for one production request. Provider-side weight changes under the same model id are outside this ledger. The fingerprint covers what the repository controls.

## Why

### Why the two failures happen together

Spend and drift are the same missing comparison. Without a fingerprint of the bytes that affect the model, the only choices are "always run" and "never notice." Always-run wastes tokens on unchanged call sites. Never-notice ships a prompt or model change that the last eval never saw.

A hand-edited flag fails for a second reason. It is not a function of those bytes, so it cannot go stale when the bytes change, and it can be set without a passing run.

### What the fingerprint-and-seal split preserves

- Unchanged call sites do not call the model. A matching fingerprint refuses the live run.
- A changed call site is visible on every unit-suite run, as a warning, without requiring an API key.
- That warning cannot land on the main branch. The merge gate fails while any call site is unknown or untested.
- A machine without a key can still commit on a working branch and open a review. Someone who can run the eval updates the ledger before the change reaches main.
- A passing eval is the only writer of the ledger. A partial or failed run leaves the previous record in place.
- A seal over the records makes a hand edit obvious. The edit cannot be committed on any branch as if the eval had passed.

### Why this approach

Running the live eval inside the unit suite keeps the stamp fresh and spends tokens on every unrelated change. It also blocks contributors whose environment has no model key.

Failing the unit suite on a stale fingerprint has the same block. The warning exists so the suite can pass. The failure moves to the main-branch gate, where the eval is supposed to have been run already.

A whole-graph live run, when only one call site changed, spends tokens on the sites whose fingerprints still match. One eval command owns one call site.

Hashing the graph wiring, or every file the call module imports, marks routing and type-only edits as untested. That forces paid reruns for changes the model never sees. The fingerprint stays on the prompt, the input-building bytes, the cases, the eval runner, the shared client, and the resolved model id.

Leaving the shared client out of the hash hides a change that affects every call. Putting one call site's prompt text inside that shared client does the reverse: one site's edit invalidates every fingerprint. Prompt text stays in the call site. The shared client stays in every fingerprint.

A seal that is only a comment, or a hash someone updates by hand next to the records, does not stop a mistaken edit. The seal is computed from the records. The eval command writes both. A mismatch discards the records.

## What worked

Keep three automated layers. None of them call the model except the live eval command, and that command calls it only for the one site whose fingerprint is missing or different.

### 1. Fingerprint each paid call site

Status is one of three values:

| Status | Meaning |
| --- | --- |
| `unknown` | The ledger has no record for this call site |
| `untested` | A record exists and its fingerprint differs from the current bytes |
| `tested` | The saved fingerprint matches the current bytes |

Hash, in a stable order, the bytes that change what the model sees or how the eval scores it:

- the prompt
- the module that builds the model input
- local value imports of that module that contribute to the input (omit type-only imports)
- the case file
- the eval runner
- the shared chat client used by every paid call
- the resolved model id

When the call site uses "whatever the current default tier is," include the config that selects that default. Otherwise hash the resolved id and leave the name-mapping file out, so a comment in the mapping file does not force a rerun.

Leave graph wiring out. A routing edit does not change the model input.

Sort parts by path. Bind each part as path, byte length, and text, so two different files cannot concatenate into the same hash. Append the model id the same way.

A unit test keeps the list honest without spending tokens: every paid call belongs to exactly one listed module, a new local value import is either listed for that site or on a short plumbing allowlist, and the declared tier matches the graph registration. Plumbing on the allowlist (validation, prompt loading, error helpers, config, model-name lookup) stays out of the per-site input list. The shared client is allowlisted as an import and still hashed once for every site, through the shared-client rule above.

### 2. Compare, then warn or refuse

On every unit-suite run, recompute fingerprints and print a warning for each site that is `unknown` or `untested`, and for each ledger id that is not a paid call site. The suite still passes.

| Check | Unknown, untested, or stray id | Fingerprint matches | Seal does not match |
| --- | --- | --- | --- |
| Unit suite | Warning, suite passes | Silent | Warning, suite passes |
| Commit on a working branch | Allowed | Allowed | Blocked |
| Commit on the main branch | Blocked | Allowed | Blocked |
| Hand-run ledger check | Fails | Passes | Fails |
| Live eval | Runs that one site | Refuses unless forced | Refuses, and does not write |

The live command:

1. If the seal does not match, exit with an error. Do not call the model. Do not write the ledger.
2. If this site is `tested` and the command was not given an explicit force flag, exit with an error. Do not call the model.
3. Otherwise run only this site's cases.
4. If every case passed and the total is greater than zero, write this site's fingerprint, pass count, and total, then recompute the seal. Any other result leaves the file unchanged.

Force is for a deliberate rerun of a site that is already tested. It is not a way to overwrite a broken seal.

### 3. Seal the ledger so the fingerprints cannot be mistyped

Store the records and a seal in the same file. The seal is a hash of the records in canonical form: ids sorted, fields in a fixed order. Key order in the file does not change the seal. A changed fingerprint, pass count, or total does.

Only the passing eval writes the file, and it writes the records and the seal together.

If the stored seal is not the hash of the records, treat the file as hand-edited:

- Ignore every record. Every paid site is `unknown`.
- Say the file was edited by hand and should be restored from version control.
- Block the commit on every branch, including working branches.
- Refuse the live eval, including a forced one, so a broken file is not "fixed" by running the model on top of discarded records.

A formatter that only reorders keys still matches. A person who pastes a new fingerprint and leaves the old seal does not.

Record shape:

```json
{
  "nodes": {
    "example-call": {
      "contentHash": "<fingerprint of that call site>",
      "passed": 3,
      "total": 3
    }
  },
  "seal": "<hash of the canonical node records>"
}
```

`passed` and `total` are the last full run. They are inside the seal so a hand edit of the score breaks it too. They are not the fingerprint. The fingerprint is `contentHash`.

## What did not work

- A hand-edited tested flag. It stays set after the prompt or the input builder changes, so the ledger and the code diverge with no warning.
- Failing the unit suite as soon as a fingerprint is stale. Accurate, and it blocks a commit from a machine that cannot run the eval. The warning-plus-main-branch-gate split is what lets that machine still publish the change for someone who can.
- One live command that walks every paid site in a graph. Sites whose fingerprints already match still spend tokens.
- Hashing only a prompt path, or only a model name written next to the call. The input builder, the cases, the eval runner, and the shared client can change the result while that narrow hash stays put.
