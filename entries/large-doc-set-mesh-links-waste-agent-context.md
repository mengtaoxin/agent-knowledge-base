---
title: Large doc-set mesh links waste agent context
tags: [documentation, markdown, linking, agents, heuristic]
agents: [all]
symptoms:
  - Markdown docs in a large set cross-link in every direction
  - Agents follow upward links and re-read high-importance docs already in context
  - Doc navigation feels like a mesh instead of a clear importance order
updated: 2026-11-02
---

# Large doc-set mesh links waste agent context

## Problem

In a repository with many Markdown documents, authors often add mutual links so editors and coding agents can jump between related pages. When those links form a mesh — every useful page points at every other useful page, including back to the always-read entry docs — agents follow the upward edges and spend context on material they have already seen in the same task. Humans also lose a clear sense of which page is the starting point and which pages are detail.

The cost shows up mainly when the doc set is large (on the order of a hundred files or more). A project with only a handful or a dozen docs rarely needs a linking rule at all.

## Context

Any project that keeps a large agent-facing or human-facing Markdown tree, and that uses relative links between those files for jump-to reading. The lesson is about link *direction* among docs of different importance, not about a specific Markdown flavor or agent product.

This is a soft heuristic, not a complete methodology and not a design pattern. Prefer the project's existing habits and written rules when they already settle how docs should link. Skip the heuristic when the tree is small enough that a mesh is still cheap to hold in mind.

## Why

### Why mesh links waste context

Coding agents often enter through high-importance docs (always-on rules, short maps, default workflows). From there they follow links into mid- and low-importance detail. If those detail pages point back “up,” the agent is invited to open the same high-importance material again. The prior turn and history already carry that content; the upward link mostly re-loads it and crowds out room for the detail the task still needs.

### What importance-downward linking preserves

- **Cheaper agent reads.** Mid-importance pages are usually reached from high-importance ones, so the agent already has the upstream context and does not need a return link to recover it.
- **A clearer tree.** Links mostly flow with importance: high → mid → low. Same-importance pages may still point at each other. The graph stays navigable instead of a dense mesh.
- **Optional adoption.** The rule is light enough to apply only where scale makes mesh costly, without forcing small doc sets into the same discipline.

### Why this approach, not the obvious alternatives

- **Fully connected cross-links “for discoverability.”** Easy to add page by page; hard to prune. Discoverability becomes re-read noise for agents that already hold the entry docs.
- **No cross-links at all.** Avoids mesh cost, but editors and agents lose cheap jumps into the right mid- or low-importance page from a short map.
- **Treating the rule as a mandatory best practice or a design pattern.** Overstates completeness. Many small or mid-size trees are fine without it; project style still wins.
- **Enforcing a strict hierarchy with no same-level links.** Unnecessary. Peer pages at the same importance can usefully reference each other without creating upward re-reads.

The matching property is **importance-ordered discovery cost**: point downward (and sideways among peers) so the next read is new detail, not a second pass over docs the agent already opened.

## What worked

Treat Markdown cross-links in a *large* doc set as an **importance-downward link-flow heuristic** (one-way flow, like water from high to low — not a mesh).

**Importance bands (rough probabilities of being opened for a typical task):**

- **High:** opened for roughly 95% of tasks (always-on rules, short entry maps).
- **Mid:** opened for roughly 20%–50% of tasks.
- **Low:** opened for under roughly 10% of tasks.

**Link direction:**

- High may link to mid; mid may link to low.
- Same band may link both ways (high↔high, mid↔mid, low↔low).
- Prefer not to link low → mid, mid → high, or low → high when the only reason is “related”; the reader usually already came from above.

**When to ignore the heuristic:**

- The project has only a few or a few dozen docs.
- Existing project conventions already define a different link style.
- A rare upward pointer is needed for a one-off escape hatch; keep those exceptional and sparse.

Illustrative shape (names only):

```text
AGENTS.md  (high)  -->  docs/workflows.md  (mid)  -->  docs/edge-cases/foo.md  (low)
     |                        |
     v                        v
 rules/*.md (high)       docs/workflows-bar.md (mid)
```

## What did not work

- Adding reciprocal links on every related pair so “any page can reach any other page.”
- Applying the same strict one-way rule to a small doc set where a mesh is still easy to maintain.
- Framing the idea as a required best practice or a heavyweight design pattern; that oversold a soft, incomplete heuristic.

## References

- Related lesson on keeping docs thin and pointer-heavy: [docs-that-mirror-code-drift-and-become-noise.md](docs-that-mirror-code-drift-and-become-noise.md)
