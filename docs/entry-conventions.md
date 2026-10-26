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
2. `## Problem` — what failed, in one or two paragraphs
3. `## Context` — tools, libraries, versions, and where it showed up
4. `## Why` — the mechanism, the benefit of the fix, and why this approach was chosen
5. `## What worked` — the fix or conclusion, with the smallest reusable illustration
6. `## What did not work` — attempts that looked right and still failed; omit if none
7. `## References` — links or doc titles; omit if none

### Problem

State the failure clearly. Prefer the user's symptoms over abstract theory.

### Context

Name the environment that matters: framework, library versions, runtime, and the surrounding workflow. Skip details that do not change the diagnosis.

### Why

An entry outlives the chat that produced it. The next person who edits it will not have that conversation, so the entry has to carry the reason for the fix.

Write three things:

- **Why the failure happens.** Name the mechanism behind the symptom. A later edit should be able to tell whether a new tool version still hits the same cause.
- **What the fix preserves.** State the benefit of solving it this way: which constraint stays intact, which cost goes away, or which later step stays reliable.
- **Why this approach.** Name the plausible alternatives — another skill, an in-conversation plan, a one-off prompt, a project-local copy of a procedure — and say what each one leaves unresolved. The point is the property that makes this choice match the cause.

`## What worked` stays the action to take. `## What did not work` stays the attempts that were actually tried and still failed. `## Why` may discuss alternatives that were not tried, when the choice of method needs a reason on record.

Do not turn `## Why` into a second procedure. Do not paste steps from a skill or a doc. Link that source and explain the properties that make it the right one.

A few short paragraphs is enough when the mechanism is simple. Go deeper when a later maintainer could reasonably pick a different method and miss the cause. Skip details that do not change the choice.

### What worked

Put the transferable answer here. Keep code or command examples minimal. A fenced code block is an illustration inside the entry, not a program to run from this repository.

### What did not work

Record false leads that waste time later. One short bullet or paragraph each is enough.

### References

Link upstream docs, issues, or changelogs that support the conclusion.

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

What failed, in one or two paragraphs.

## Context

Tools, libraries, versions, and where it showed up.

## Why

Why the failure happens, what the fix preserves, and why this approach matches the cause better than the alternatives.

## What worked

The fix or the conclusion, with the smallest illustration that makes it reusable.

## What did not work

Attempts that looked right and still failed. Omit this section if there were none.

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
- [ ] Symptoms are exact enough for another agent to match
- [ ] Every load-bearing fact was confirmed with the user; nothing was filled in by default or by guesswork
- [ ] Code blocks are documentation only
- [ ] No secrets or credentials
