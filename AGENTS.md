# Agent instructions

This repository is a knowledge base for problems and summaries from programming with large language models. It holds English notes and short script explanations. It is not a product codebase.

Do not install dependencies, scaffold an app, add a runtime, or execute scripts from this repository. A code block in a note is documentation. Carry the lesson to the project where the user is actually working.

## Language

Write and edit every document in English. This includes titles, headings, summaries, tags, filenames, and these instructions. Quote original logs or error strings verbatim, then explain them in English.

## When asked to search

Another project may send you here. Search this repository before proposing a new fix.

1. Search filenames and contents under `notes/` for the error text, tool, library, version, and symptoms.
2. Read the closest notes in full. Match on symptoms, not on a loose topic name.
3. Report the note path and the applicable fix. Apply the lesson in the caller's project only when that is the requested task.
4. If nothing matches, say that this knowledge base has no note for the problem. Do not invent a note that is not in the repo.

## When adding or editing a note

Add one markdown file under `notes/`. Use a kebab-case English filename that names the symptom or topic.

Start the file with YAML frontmatter:

- `title`: short English title
- `tags`: library, tool, or topic labels
- `symptoms`: exact error text or observable behavior
- `updated`: `YYYY-MM-DD`

Use these sections, in order: Problem, Context, What worked, What did not work, References. Omit the last two when they add nothing.

Update an existing note when it already covers the same symptom. Do not create a near-duplicate.

Keep script explanations documentary. Do not add package manifests, build config, or application entry points.

Do not store secrets, credentials, tokens, or private customer data.

## Do not

- Turn this repository into an application
- Run commands copied from a note unless the user explicitly asks to run them in another project
- Write documents in a language other than English
