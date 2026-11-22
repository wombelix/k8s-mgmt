<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0005: Local Ansible collections

## Status

Accepted

Date: 2026-10-24

## Context

The playbooks need collections beyond the Ansible built-ins. Installing
them globally makes runs depend on the machine and mixes versions
between projects.

## Decision

Ansible collections are installed into the repo from pinned versions,
not globally.

- The Ansible config points the collection and role paths to folders in
  the repo.
- A requirements file lists the collections with pinned versions. A
  requirements file for roles follows once a role is needed.
- One task installs them, see
  [ADR-0006](0006-taskfile-as-command-interface.md).
- Git tracks only the requirements files, not the installed content.

## Consequences

- Every clone and the pipeline use the same versions.
- The global Ansible install stays clean.
- ansible-lint finds the requirements file without extra config.
- Each clone needs one install run before the first playbook run.
- A version bump is a change in the repo. Renovate opens these changes,
  see [ADR-0019](0019-dependency-updates-with-renovate.md).

## Alternatives Considered

- A global install needs no step per clone. It depends on the machine,
  and versions drift.
- Committing the installed collections needs no install step. It puts
  large third-party code in the repo.

## Related

- Depends on: [ADR-0006](0006-taskfile-as-command-interface.md)
- Amended by: [ADR-0019](0019-dependency-updates-with-renovate.md)
