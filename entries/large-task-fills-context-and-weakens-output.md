---
title: Large task fills one context and weakens later output
tags: [context, tasks, agent-kit]
agents: [all]
symptoms:
  - One conversation handles a large task from start to finish
  - Later steps forget earlier constraints, files, or decisions
  - The agent can still finish, but the result drifts from the request
  - Context is crowded by exploration, tool output, and intermediate edits
updated: 2026-10-26
---

# Large task fills one context and weakens later output

## Problem

A request is large enough that one conversation can still complete it. Keeping the whole job in that conversation still hurts the result. Exploration, tool output, failed attempts, and intermediate edits accumulate. Later steps remember less of the original requirement, and the final artifact is weaker or off-spec.

The goal of splitting is to keep each context small. It is not a claim that the agent is unable to finish the task in one pass.

## Context

Any coding agent with a finite context window. The lesson applies when the work can be split into serial pieces, each with a bounded scope, that depend only on earlier pieces.

How to split, track, and resume the work lives in the `task-divide-and-conquer` skill in [agent-kit](https://github.com/mengtaoxin/agent-kit/tree/main/skills/task-divide-and-conquer). Read that skill for the procedure. This entry records when to use it, and why that shape of split matches the failure.

## Why

### Why the later steps get worse

A coding agent's context is a finite window, and a large task fills it with more than the request. Exploration, tool output, diffs, failed attempts, and intermediate plans all stay in the window for every later step. The original constraints then compete with whatever was produced most recently. The agent can still emit a finish. What drops is fidelity: a file named at the start, a constraint stated once, or a decision that was already settled.

Splitting is worthwhile because it changes what sits in that window.

- Each implementation step starts in a fresh context. Its working set is the written requirement, the scope of that one step, and the short results of earlier steps. The constraints for that step stay near the front of the context.
- Exploration and failed attempts from one step are not replayed into the next. A wrong turn stays in the context that made it.
- The plan lives in a file, outside any transcript. Stopping in the middle leaves a recoverable state, and resuming does not depend on the previous chat still being open or still accurate.
- The conversation that launches the steps stays small too, as long as it reads the file after each step and does not absorb the step's transcript.

A summary written inside the same conversation does not provide those properties. The summary is produced by a context that is already crowded, and the transcript that produced it is still there.

### Why this skill

The failure is accumulated context, so a split only works when the next step does not inherit the previous transcript. `task-divide-and-conquer` is built around that property. One parent task becomes serial subtasks, one subtask runs in one isolated context (a sub-agent, or a new conversation when the host cannot launch one), and a markdown file under `.task-divide-and-conquer/` is the only shared state. The requirement stays written down verbatim. Each subtask's `Result` carries what the next step must know, not the exploration that produced it. The orchestrator chooses the next step by re-reading the file. It does not treat the executor transcript as the plan, which is what would pull the large context back into the parent.

Other ways of splitting leave that cause in place:

- **A checklist in the same conversation.** The list records remaining work. It does not discard the exploration that already filled the window, so the next step still runs crowded.
- **A new conversation plus a pasted recap.** The recap is an informal summary. Acceptance criteria, files, and decisions drop out or get rewritten, and the new context spends its budget reconstructing the plan.
- **Parallel sub-agents.** Later pieces of this kind of task depend on earlier results: a file the previous step created, a decision recorded in its result. Parallel runs race those dependencies and produce conflicting edits. The skill requires a serial order, and each subtask may depend only on earlier subtasks.
- **The parent conversation implements the pieces itself.** The parent then accumulates every transcript. The skill keeps the orchestrator from implementing later subtasks, and keeps each executor to exactly one subtask.
- **A copy of the procedure in the project** (a local rule, prompt, or template). The copy becomes a second source of truth and drifts when the skill changes. The project keeps the task file, which is the state for that one job, and follows the skill file for the procedure.
- **Planning and the first code change in one chat.** The exploration used to discover the split is the bulk that should not sit in the implementation context. The skill writes the plan into the task file first, then starts implementation in a context that only reads the file.

Two further properties matter when the plan itself is wrong, or when the host has no sub-agents. A bad split stops the chain and records a failure reason, instead of a quiet redesign inside a context that has already started to drift. A host that cannot launch sub-agents uses the same file and a new conversation, so isolation does not depend on a sub-agent feature.

This entry does not restate the skill's modes or template. Those stay in the skill so the procedure has one source.

## What worked

When a task is large enough that one context would crowd out earlier constraints, hand it to `task-divide-and-conquer`.

The skill splits the request into a parent task and serial subtasks, and runs one subtask per isolated context: a sub-agent when the host can launch one, otherwise a new conversation. A markdown task file under `.task-divide-and-conquer/` is the only shared state between those contexts.

Do not copy the skill's modes, file template, or prompts into the project. Follow the skill file.

## What did not work

- Finishing the large task in one conversation because it is still possible. The context still grows, and later output degrades.
- Splitting the work without a shared task file. The next context then has to reconstruct the plan from the previous transcript, which puts the bulk of the context back.

## References

- Skill `task-divide-and-conquer`: [agent-kit/skills/task-divide-and-conquer](https://github.com/mengtaoxin/agent-kit/tree/main/skills/task-divide-and-conquer)
