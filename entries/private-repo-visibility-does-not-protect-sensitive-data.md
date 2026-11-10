---
title: Private repo visibility does not protect sensitive data
tags: [git, repositories, data-security, secrets, multi-repo]
agents: [all]
symptoms:
  - Team maintains dozens or hundreds of repos, some public and some private
  - Private repo content appears in public issues, PRs, or cross-repo references
  - Sensitive files and non-public datasets live in a private git remote as the main store
  - After a leak, many local clones still hold full copies that are hard to recall
updated: 2026-11-10
---

# Private repo visibility does not protect sensitive data

## Problem

In a multi-repository workplace, developers often maintain dozens or hundreds of remotes. Some remotes are public; some are private. Private visibility feels like a boundary, but it is not a data-security boundary.

A private repository that depends on a public one creates a daily coupling surface: issues, pull requests, commit messages, dependency notes, and casual cross-links. Non-public details travel across that surface by accident. Collaboration multiplies the copies: every clone is a full working tree on someone else's disk. Once material has left the intended circle, git history and those clones make cleanup slow and incomplete.

Relying on “the repo is private” to protect non-public data therefore fails in ordinary maintenance, not only in dramatic breach scenarios.

## Context

Any org or personal setup that mixes public and private git hosts (GitHub, GitLab, or similar), with frequent cross-repo work and many local clones. The lesson is about where non-public data may live, not about a specific host product or version.

Typical pressure: private remotes already hold large trees of files that were never meant for a public surface. Moving that material out is expensive. That cost is real; it does not make private visibility a substitute for separating secrets and non-public payloads from source control.

## Why

### Why private visibility fails as confidentiality

Private remotes control who can browse the hosted tree by default. They do not control:

- What people paste into public issues and PRs when debugging a shared dependency
- What agents and humans quote when a private repo and a public repo are open in the same workflow
- How many full clones exist on laptops and CI runners
- Whether a future ACL mistake, fork, or accidental push makes the remote readable more widely

Git is a distribution and history system. Treating “private” as “safe to store anything we must not publish” confuses access control with data classification.

### What the preferred split preserves

- **A clear rule for new work:** code and docs that can be public should prefer public remotes; anything that must stay non-public does not enter git at all.
- **A durable store for non-public payloads:** databases, private object storage, or a private file server hold the real confidential material, with access controlled outside the clone graph.
- **Honest expectations for existing private remotes:** large historical trees that cannot be opened today are understandable. The norm is directional, not a demand for an overnight cutover of every private archive.

### Why this approach, not the obvious alternatives

- **Keep everything private and trust ACL forever.** Access mistakes, cross-repo chatter, and local clones still leak. Privacy settings are one control, not the data plane.
- **Public when possible, but keep non-public datasets in private repos “because only our team can clone.”** That still ships full copies to every collaborator and still invites cross-link leakage into public surfaces.
- **Mandate an immediate migration of every private remote to a pure-public + external-storage model.** Correct as a long-term shape, unrealistic as a single switch. Existing private remotes often already contain too much to publish; forcing a one-shot open or move creates rejection and unfinished half-migrations.
- **Treat private remotes as if they were already public for classification, while allowing gradual cleanup.** New sensitive material stays out of git. Legacy trees can remain private while teams peel non-public data into proper stores over time. That matches both the security mechanism and the cost of reality.

The matching property is **classification independent of ACL**: assume any clone may eventually be read more widely than today's private setting implies, and put truly non-public data where git is not the system of record.

## What worked

Use this operating rule:

1. Prefer public remotes when the content can be public. Open code is easier to collaborate on and removes false confidence from a private flag.
2. Treat every private remote as if it were public for data classification. Do not put core non-public data into git, including into private remotes.
3. Keep secrets, customer payloads, proprietary datasets, and other non-public files in a database, a private bucket, or a private file server. Reference them from code; do not commit them.
4. When reviewing agent or human changes in a mixed public/private workspace, refuse to copy private-repo specifics into public issues, PRs, or public remotes.
5. For legacy private remotes that already hold large non-public trees: keep them private for now if opening them is not feasible, and still stop adding new non-public material. Migrate sensitive payloads out when the cost is justified; do not pretend the private flag is enough, and do not demand a single cutover that the org cannot finish.

Practical check before committing: “Would I be willing for this path to appear in a public fork tomorrow?” If no, it does not belong in the repository.

## What did not work

- Treating private repository visibility as the primary confidentiality control for datasets and private files.
- Assuming that because a dependency link is “internal,” public issues and PRs will never carry private context.
- Planning only for “make every repo public someday” without a place outside git for non-public data — the migration then stalls, and sensitive material stays in clones indefinitely.
- Demanding an all-at-once purge of every legacy private tree. Teams correctly push back when the archive is large and the open/migrate cost is high; the better path is a firm rule for new data plus gradual extraction.

## References

None beyond ordinary git-host ACL and secret-handling practice. The lesson is organizational classification, not a single product doc.
