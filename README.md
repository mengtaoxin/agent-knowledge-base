# Agent Knowledge Base

A searchable library of notes from programming with large language models. It stores problems, diagnoses, and short explanations of scripts. It is not an application. Nothing in this repository is meant to be installed or executed.

When a problem shows up in another project, point an agent at this repository and ask it to search here before proposing a new fix.

## Language

Write every document in English. Titles, headings, summaries, tags, and explanations stay in English so agents can match them reliably. Quote original logs and error strings exactly, then explain them in English.

## What to store

- A problem encountered while programming with a large language model
- The context: tool, library, version, and the exact error or behavior
- What was tried, and what actually worked
- A short explanation of a script or command, kept as documentation

Leave out application source code, secrets, credentials, and notes that have no lesson beyond one private repository.

## Layout

```text
notes/
  <topic-slug>.md
```

One note per file. Name the file in kebab-case English after the symptom or topic, for example `next-middleware-matcher-skips-api.md`.

## Note template

```markdown
---
title: Short English title
tags: [library-or-tool, topic]
symptoms:
  - Exact error text or observable behavior
updated: YYYY-MM-DD
---

# Short English title

## Problem

What failed, in one or two paragraphs.

## Context

Tools, libraries, versions, and where it showed up.

## What worked

The fix or the conclusion, with the smallest illustration that makes it reusable.

## What did not work

Attempts that looked right and still failed. Omit this section if there were none.

## References

Links or doc titles. Omit this section if there are none.
```

A fenced code block is an illustration inside a note. It is not a program to run from this repository.

## How to search from another project

Ask the agent to search `notes/` for the error text, library name, and symptoms. A useful request names the failure and points at this repo:

```text
Search the agent-knowledge-base repo for notes about: <symptom, library, and error text>.
```

The agent should return the matching note path and the applicable fix, then apply that lesson in the other project. If no note matches, it should say so.
