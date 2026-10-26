# File structure

This repository stores documentation only. It has no application source, package manifest, or runtime.

```text
README.md
AGENTS.md
docs/
  file-structure.md
  entry-conventions.md
entries/
  <symptom-slug>.md
```

## Top level

| Path | Role |
| --- | --- |
| `README.md` | Short introduction to what this repository is for |
| `AGENTS.md` | Instructions for agents that search or edit this repository |

## `docs/`

Longer descriptions of how the repository is organized and how to write entries.

| Path | Role |
| --- | --- |
| `docs/file-structure.md` | This file. Directory roles and where new files go |
| `docs/entry-conventions.md` | How to name, structure, and update an entry |

Add another file under `docs/` when a convention is too long for `README.md` or `AGENTS.md`.

## `entries/`

One markdown file per lesson. The filename is kebab-case English and names the symptom clearly enough to tell this entry from the others. A longer name is appropriate when a shorter one would be ambiguous or easy to confuse with another file.

```text
entries/
  next-middleware-matcher-skips-api.md
```

Writing rules for these files are in [entry-conventions.md](entry-conventions.md).

## Where a new file goes

- A problem, diagnosis, or script explanation goes in `entries/`.
- A description of repository structure or writing rules goes in `docs/`.
- Do not add application code, dependency files, or build configuration.
