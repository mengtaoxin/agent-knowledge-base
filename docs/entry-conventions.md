# Entry conventions

How to write entries under `entries/`. Follow this file when adding or editing an entry. Write every entry in English.

## Purpose of an entry

An entry records one reusable lesson from programming with a large language model. It should help an agent in another project recognize the same failure and apply the same fix.

Store:

- A problem and its diagnosis
- The tool, library, version, and exact error or behavior
- What was tried, and what worked
- Why that fix is the one that matches the cause, including why the obvious alternatives do not
- A short explanation of a script or command, kept as documentation

Recommended. Write these in as much detail as the author can give. An entry is complete without them:

- A real case
- The author's own judgment, including a personal or subjective one
- The path of thought, and the rule as it was revised
- The thoughts, mood, and feeling at the time of writing

Do not store:

- Application source code meant to be run from this repository
- Secrets, credentials, tokens, or private customer data
- Entries that only make sense inside one private repository and have no transferable lesson

## File location and naming

```text
entries/
  <symptom-slug>.md
```

- One entry per file
- Filename in kebab-case English
- Name the file after the observable symptom, so a later search can tell this entry from every other entry

Clarity comes before brevity. A few extra words are worth a longer filename. Keep a word when dropping it would make the name hard to understand, or easy to confuse with another entry.

Good: `cursor-playwright-redownloads-browsers-every-run.md`

Also good: `next-middleware-matcher-skips-api.md`

Too vague to tell entries apart: `playwright-browsers.md`, `next-entries.md`, `misc.md`, `fix.md`

`playwright-browsers.md` could be a cache path, a CI install, or a different agent. The longer name says which product re-downloads browsers, and that it happens on every run.

## Frontmatter

Every entry starts with YAML frontmatter:

| Field | Required | Description |
| --- | --- | --- |
| `title` | Yes | Short English title; usually matches the H1 |
| `tags` | Yes | Library, tool, or topic labels for search |
| `agents` | Yes | Which coding agents the lesson applies to |
| `symptoms` | Yes | Exact error text or observable behavior |
| `updated` | Yes | Last meaningful edit date as `YYYY-MM-DD` |

Example:

```yaml
---
title: Next.js middleware matcher skips API routes
tags: [next.js, middleware]
agents: [all]
symptoms:
  - Middleware never runs for /api/*
  - matcher config looks correct but requests bypass it
updated: 2026-10-25
---
```

### Field guidance

- `title`: one line, searchable, specific to the failure
- `tags`: lowercase when practical; prefer real product or library names
- `agents`: a YAML list of coding-agent names the entry applies to. Use `[all]` when any coding agent can reuse the lesson. Use specific product names when the failure or fix only exists in that agent (for example `[cursor]` for Cursor Agent sandbox behavior). Prefer lowercase product slugs: `cursor`, `claude-code`, and so on. Do not mix `all` with other names.
- `symptoms`: quote exact error strings when they exist; otherwise describe the observable behavior in English
- `updated`: change this whenever the lesson or fix changes

## Body sections

Use these headings in this order:

1. `# Short English title` — same idea as `title` in frontmatter
2. `## Problem` — what failed. A real episode is recommended when one is known
3. `## Context` — tools, libraries, versions, and where it showed up
4. `## Why` — the mechanism, the benefit of the fix, and why this approach was chosen
5. `## What worked` — the fix or conclusion, with the smallest reusable illustration
6. `## What did not work` — attempts that looked right and still failed; omit if none
7. `## References` — links or doc titles; omit if none

### Problem

State the failure clearly. Prefer the user's symptoms over abstract theory. When a real episode is known, include it, at the length the case takes. The entry still stands without that episode.

### Context

Name the environment that matters: framework, library versions, runtime, and the surrounding workflow. When the situation around the case is known, include it, so a later reader can picture where the judgment was made. That situation is recommended detail, and the entry still stands without it.

### Why

An entry outlives the chat that produced it. The next person who edits it will not have that conversation, so the entry has to carry the reason for the fix.

Write three things:

- **Why the failure happens.** Name the mechanism behind the symptom. A later edit should be able to tell whether a new tool version still hits the same cause.
- **What the fix preserves.** State the benefit of solving it this way: which constraint stays intact, which cost goes away, or which later step stays reliable.
- **Why this approach.** Name the plausible alternatives — another skill, an in-conversation plan, a one-off prompt, a project-local copy of a procedure — and say what each one leaves unresolved. The point is the property that makes this choice match the cause.

`## What worked` stays the action to take. `## What did not work` stays the attempts that were actually tried and still failed. `## Why` may discuss alternatives that were not tried, when the choice of method needs a reason on record.

Do not turn `## Why` into a second procedure. Do not paste steps from a skill or a doc. Link that source and explain the properties that make it the right one.

Write the mechanism, the benefit, and the choice at the depth a later maintainer needs. Those three carry the lesson. The path of the decision, each version the rule passed through, and the author's judgment and state of mind are recommended on top of them, as described in [Length and the author's account](#length-and-the-authors-account).

### What worked

Put the transferable answer here. Keep code or command examples minimal. A fenced code block is an illustration inside the entry, not a program to run from this repository.

### What did not work

Record false leads that waste time later. Tell each one as it happened: what it looked like, why it was tried, and how it failed. A bullet is enough for a small lead. A longer account is appropriate when the attempt was part of how the rule changed.

### References

Link upstream docs, issues, changelogs, or related entries that support the conclusion for a reader in another project. Do not link this repository's authoring docs under `docs/` (for example `entry-conventions.md` or `file-structure.md`); those exist only to search and edit this knowledge base.

## Length and the author's account

The items below are recommended. More detail is better. An entry that already states the problem, the cause, and the fix is complete without them, and a missing item does not block saving.

When the author has any of this, write it in, and write it as fully as they gave it:

- A real case: what was happening, what was tried, and what changed
- The author's own judgment, including a personal or subjective one
- The path of thought, and the rule as it was revised: the earlier wording, what felt wrong about it, and what the next version kept
- The thoughts, the mood, and the feeling at the time of writing, in the most honest statement of that state the author can give

That material may be written as a story. When the path needs the room, write it at the length of a novella. A later reader can then follow what the author was thinking and feeling while the entry was written, and meet that state as it actually was.

Keep the required sections. When this material is present, put the case in `## Problem` and `## Context`. Put the path, the revisions, the judgment, and the feeling in `## Why`. Put attempts that were actually tried in `## What did not work`. Leave `## What worked` as the action a later agent can apply.

Keep what the author says. Do not invent a case, a judgment, a revision, or a state of mind to fill a gap. Leave the entry as it is when the author has not given that part.

## Full template

```markdown
---
title: Short English title
tags: [library-or-tool, topic]
agents: [all]
symptoms:
  - Exact error text or observable behavior
updated: YYYY-MM-DD
---

# Short English title

## Problem

What failed. A real episode is welcome when one is known.

## Context

Tools, libraries, versions, and where it showed up. The situation around the case is welcome when it is known.

## Why

Why the failure happens, what the fix preserves, and why this approach matches the cause. Recommended in addition, when the author has given them: the path of the decision, the versions the rule passed through, and the author's judgment, thoughts, and feelings at the time of writing.

## What worked

The fix or the conclusion, with the smallest illustration that makes it reusable.

## What did not work

Attempts that looked right and still failed, told as they happened. Omit this section if there were none.

## References

Links or doc titles. Omit this section if there are none.
```

## Editing rules

- Prefer updating an existing entry when it already covers the same symptom
- Do not create a near-duplicate with a different filename
- Keep script explanations documentary; do not add package manifests, build config, or application startup files
- Quote original logs and error strings verbatim, then explain them in English
- Keep the whole entry in English: title, headings, body, tags, and filename

## Resolve unclear points before writing

An entry is kept for later reuse. An unclear fact becomes a wrong lesson.

Before adding or editing an entry, settle anything the source leaves open: which product, which version, which symptom, whether an existing entry already covers it, which agents it applies to, why this fix, and which words the filename needs so it will not be confused with another entry. Make those points explicit. Leave no load-bearing gap filled by a default or a guess. `agents: [all]` is one such default: use it only when the lesson really applies to every coding agent.

Ask the user. When a dialog or other structured question is available, use it. Prefer a yes/no question, or labeled options such as A/B/C, so the answer picks one meaning. When a word could refer to more than one thing, ask what it refers to.

Wait for the answer before writing that part. Do not save a vague draft in the meantime.

If the meaning still cannot be made definite, do not add or edit the entry. Tell the user it does not meet this knowledge base's requirements yet, and name the point that is still unclear.

## Checklist before saving

- [ ] Filename is kebab-case English under `entries/`, and specific enough to distinguish this entry; extra words stay when they carry that distinction
- [ ] Frontmatter includes `title`, `tags`, `agents`, `symptoms`, and `updated`
- [ ] `agents` is `[all]` or a list of specific agent product slugs, not both
- [ ] Body uses the required section order, including `## Why`
- [ ] `## Why` names the mechanism, the benefit of the fix, and why this approach was chosen
- [ ] Recommended detail the author supplied (a case, a judgment, a revision, thoughts, mood, or feeling) is kept; its absence is acceptable, and none of it was invented
- [ ] Symptoms are exact enough for another agent to match
- [ ] Every load-bearing fact was confirmed with the user; nothing was filled in by default or by guesswork
- [ ] Code blocks are documentation only
- [ ] No secrets or credentials
