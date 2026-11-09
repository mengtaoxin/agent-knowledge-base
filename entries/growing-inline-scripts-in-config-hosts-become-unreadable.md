---
title: Growing inline scripts in config hosts become unreadable
tags: [github-actions, dockerfile, yaml, shell, html, agents, readability]
agents: [all]
symptoms:
  - GitHub Actions workflow YAML contains a long run: script instead of a referenced .sh or JS file
  - A Dockerfile packs many shell statements into one RUN for layer caching and becomes hard to follow
  - An agent keeps expanding inline shell or JS inside YAML or a Dockerfile instead of extracting a script file
  - A host file mixes substantial guest-language logic and loses syntax highlighting and easy maintenance
updated: 2026-11-10
---

# Growing inline scripts in config hosts become unreadable

## Problem

Host files such as GitHub Actions workflows, Dockerfiles, and HTML can embed a guest language (shell, JavaScript, CSS). A line or two of inline script usually finishes the job. Once that guest logic grows — especially a shell script past roughly ten lines — the host file turns into an opaque blob. Readers cannot tell what the workflow, image build, or page is doing without decoding escaped newlines, YAML block scalars, or `RUN` chains written for cache layers rather than for humans.

With a coding agent, the failure gets worse. If the default is to write the script inline, the model keeps extending that block. It rarely stops and extracts a separate file on its own, because the host format already “allows” more lines.

## Context

Common workplaces for this pattern:

1. **GitHub Actions** — YAML workflows can put shell or JS directly under `run:` instead of calling an external script.
2. **Dockerfile** — `RUN` can hold arbitrary shell. To avoid many cache layers, authors often join many commands into one line, which makes a long script even harder to read than the same logic in a `.sh` file.
3. **HTML** — the long-standing practice is not to dump substantial JavaScript or CSS into the page, but to link separate files. That case is the same shape: a host document with a guest language.

The first two show up often in day-to-day CI and packaging work. The rule is not “never put code in a config-like file.” Short inline snippets are fine. The rule kicks in when the guest logic expands.

## Why

### Why the host gets unreadable

A workflow YAML or Dockerfile is optimized for orchestration and image layers, not for editing a multi-step shell program. Escaping, folding, and one-line `RUN` chains hide structure that a normal `.sh` file would show with indentation, comments, and editor support. The same holds for large inline JS or CSS inside HTML.

### What extracting preserves

- **Readable orchestration.** The host file stays a short map: checkout, call script, deploy. The script file owns the branching and loops.
- **Cheap maintenance.** Changing business logic does not mean re-parsing YAML escapes or Dockerfile line continuations.
- **Editor support.** A standalone `.sh` or `.js` file gets normal syntax highlighting and linting for humans and for agents that read the file.

### Why this approach

Telling agents “never write code in config” is too absolute and fights real one-liners. Telling them “do not mix languages in one file” misses the point: a two-line `run:` is mixed languages and still fine. The matching rule is a growth threshold: keep tiny embeds; once the guest script is expanding — especially past about ten lines of shell — extract and reference. Prefer `run: ./scripts/…` or `COPY` plus an executable script over another dozen lines inside the host.

## What worked

- Allow one or two lines of inline script in the host when that is all you need.
- When the guest logic grows (rule of thumb: shell past roughly ten lines, or any block that is hard to skim in the host), move it to a dedicated `.sh` / `.js` / `.css` file and reference that file from the workflow, Dockerfile, or HTML.
- Prefer extraction early when an agent is about to keep appending to an inline block; do not wait until the blob is already unreadable.

Example shape (illustration only):

```yaml
# Prefer
- name: Publish
  run: ./scripts/publish.sh
```

```dockerfile
# Prefer
COPY scripts/build.sh /usr/local/bin/build.sh
RUN build.sh
```

## What did not work

Leaving the script inline because “the format supports it.” That works for a couple of lines and then invites the agent (and later humans) to keep growing the same block until the host file is the hardest part of the change to review.

## References

- GitHub Actions: workflow `run` steps and composite/reusable actions that call checked-in scripts
- Dockerfile best practice of keeping `RUN` readable (and using scripts when the command list is non-trivial)
- Separating HTML structure from linked JavaScript and CSS
