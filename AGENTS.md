# Agent instructions

This repository is a knowledge base for problems and summaries from programming with large language models. It holds English entries and short script explanations. It is not a product codebase.

Do not install dependencies, scaffold an app, add a runtime, or execute scripts from this repository. A code block in an entry is documentation. Carry the lesson to the project where the user is actually working.

## Language

Write and edit every document in English. This includes titles, headings, summaries, tags, filenames, and these instructions. Quote original logs or error strings verbatim, then explain them in English.

## When asked to search

Another project may send you here. Search this repository before proposing a new fix.

1. Search filenames and contents under `entries/` for the error text, tool, library, version, and symptoms.
2. Read the closest entries in full. Match on symptoms, not on a loose topic name. Prefer entries whose `agents` frontmatter is `[all]` or includes the agent you are running as.
3. Report the entry path and the applicable fix. Apply the lesson in the caller's project only when that is the requested task.
4. If nothing matches, say that this knowledge base has no entry for the problem. Do not invent an entry that is not in the repo.

## When adding or editing an entry

Follow [docs/entry-conventions.md](docs/entry-conventions.md) in full. Directory roles are in [docs/file-structure.md](docs/file-structure.md).

In short: one markdown file under `entries/`, a kebab-case English filename that states the symptom clearly (a longer name is better than one that blurs into another entry), required frontmatter (including `agents` scope), and the standard section order. In `## Why`, record why the failure happens and why this fix was chosen, not only the steps. A real case, the author's own judgment, how the rule was revised, and the thoughts, mood, and feeling at the time of writing are recommended: more detail is better, and the entry is complete without them. When the author supplies that account, keep it; it may read as a story, and it may run as long as a novella. Do not invent it. Update an existing entry when the symptom already matches. Do not store secrets.

When a fact is unclear, ask before writing. Use a dialog when one is available, with a yes/no question or labeled options such as A/B/C. Wait for the answer. Do not fill the gap with a default or a guess. If the meaning still cannot be made definite, do not write the entry. Tell the user it does not meet this knowledge base's requirements yet, and name the point that is still unclear.

## Do not

- Turn this repository into an application
- Run commands copied from an entry unless the user explicitly asks to run them in another project
- Write documents in a language other than English
