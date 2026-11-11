---
title: Hot agent instruction docs bloat with why explanations
tags: [documentation, agents.md, cursor-rules, tokens, context]
agents: [all]
symptoms:
  - AGENTS.md, always-apply rules, or docs linked from them grow long with background and root-cause prose
  - Agents reload the same explanatory paragraphs on most tasks and spend context on non-actionable text
  - Rule files mix “what to do” with long “why we added this” narratives in one always-read surface
updated: 2026-11-11
---

# Hot agent instruction docs bloat with why explanations

## Problem

Some documents are opened on most agent tasks — far more often than ordinary project docs. Typical hot surfaces:

- `AGENTS.md` (or the product’s equivalent always-on instruction file)
- Cursor rules with `alwaysApply: true` (and similar always-on rule packs in other agents)
- Core docs that those entry files point at, so the agent reads them whenever it follows the map

Authors often write a rule as both an instruction and a justification. The justification is useful for maintainers, but it is not needed on every task. Because the agent reloads the hot file in roughly 90% or more of tasks, every extra sentence of background burns tokens whether the current task needs that history or not. The failure is not “rules use tokens.” The failure is **parking low-urgency explanation on a surface that is paid on nearly every run**.

## Context

Any repository that ships always-on agent instructions: `AGENTS.md`, Cursor project rules with `alwaysApply: true`, Claude Code / similar entry files, and the small set of docs those files treat as mandatory reading. The lesson is about **what belongs on that hot path**, not about deleting institutional knowledge.

This sits beside, but is not the same as, link-direction heuristics for large doc trees and “do not mirror code in prose” guidance. Those cover how pages link and which facts docs should own. This entry covers the **What / Why split** inside the few files that almost every task already opens.

## Why

### Why hot docs inflate context

Always-on instruction files are injected or reread as repository guidance. Length on those files is a recurring tax, not a one-time onboarding cost. A paragraph that explains how a painful incident led to a rule helps the next human editor; it rarely changes what the agent must do on this turn. Leaving that paragraph in the hot file means every unrelated task still pays for it.

A rule usually has two parts:

1. **What** — the content of the rule (the behavior to follow)
2. **Why** — the root cause or background (why the rule exists)

Mixing them in one always-read file optimizes for storytelling at the cost of steady context spend.

### What splitting preserves

- **Actionable hot docs.** The always-read surface stays a short list of constraints and procedures.
- **Recoverable history.** The Why still exists in a cold document. A maintainer who needs the chain can search for explanation wording (`why`, `root cause`, incident notes) and close the loop without the agent loading that prose by default.
- **Bounded growth.** New rules can grow in the cold archive without forcing every task to re-digest the archive.

### Why this approach, not the obvious alternatives

- **Keep What and Why together “so the agent understands intent.”** Understanding intent for the current rule rarely needs the full incident narrative. The What line already encodes the constraint; the story is for humans who edit the rule later.
- **Put a “see also” link from the hot doc to the Why doc.** Even a short pointer invites the agent to follow it and pull cold prose into a hot turn. Prefer **no link** from the hot surface to the explanation archive.
- **Delete the Why entirely.** Saves tokens, but loses the maintainer’s ability to reconstruct why the rule exists when it is challenged months later.
- **Move everything to cold docs and leave the hot file empty.** Agents then rediscover constraints from the whole tree or miss them; the hot file exists so the must-follow lines stay cheap and reliable.

The matching property is **read frequency versus urgency**: put only must-follow What on high-frequency surfaces; park Why where search can find it and routine tasks do not.

## What worked

Treat hot instruction docs as **What-only**. Keep explanations in a separate, rarely opened document. Do not link from the hot doc to that explanation.

**Keep in the hot file (What):**

- The constraint or procedure in the fewest words that still work
- Only details the agent must apply on ordinary tasks

**Move out (Why):**

- Incident history, root-cause narrative, and “we chose this after …”
- Long justification of why a softer wording failed
- Background that a maintainer needs when revising or deleting the rule

**Illustrative split (browser verification ban):**

Hot doc (`AGENTS.md` or an always-apply rule) — What only:

```markdown
## Browser verification is forbidden

Never verify this application in a browser. Prove behavior only with this project's automated test commands.
```

Cold doc (no inbound link from the hot file) — Why:

```markdown
# Why browser verification is forbidden

Cursor forces a browser session after visual or layout changes. The session is slow and token-heavy. Soft wording such as "prefer tests" still loses to that default. The product page has no switch to turn the check off, so the repository instruction must state an absolute ban and name automated tests as the only proof.
```

The cold file may sit with almost no inbound links. That is fine. Search for `why` / explanation text when someone needs the chain; do not make every agent task discover it through the hot map.

**Core rule:** for documents opened on most tasks, slim them; the usual cut is to split root cause and Why out, and leave only What on the hot path.

## What did not work

- Packing root-cause essays into `AGENTS.md` or `alwaysApply` rules so “everything is in one place.”
- Using a formatting-only anecdote (for example blank lines between functions) as the motivating example; that does not show the always-read / token-tax problem as clearly as a hot agent instruction.
- Adding “see why” links from hot docs into the explanation archive, which reintroduces cold prose onto the hot path when the agent follows them.

## References

- Related lesson on importance-downward linking in large doc sets: [large-doc-set-mesh-links-waste-agent-context.md](large-doc-set-mesh-links-waste-agent-context.md)
- Related lesson on the browser-verification ban used as the What example: [cursor-agent-always-opens-browser-to-verify-visual-changes.md](cursor-agent-always-opens-browser-to-verify-visual-changes.md)
