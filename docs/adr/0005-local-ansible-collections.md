<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0005: Local Ansible collections

## Status

Accepted

Date: 2026-10-24

## Context

The playbooks need collections beyond `ansible.builtin`, for example
`ansible.posix`. Installing them globally makes runs depend on the
machine and mixes versions between projects.

## Decision

* `ansible.cfg` sets `collections_path = ./collections` and
  `roles_path = ./roles`.
* `collections/requirements.yml` lists the collections with pinned
  versions. `roles/requirements.yml` follows once a role is needed.
* `ansible-galaxy` installs them into the repo with `task deps`.
* Git tracks only the requirements files. `.gitignore` excludes the
  installed content.

## Consequences

* Every clone and the pipeline use the same versions.
* The global Ansible install stays clean.
* ansible-lint finds `collections/requirements.yml` without extra
  config.
* `reuse lint` passes only once a requirements file is tracked. Before
  that, Git does not report the installed folder as ignored.

## Alternatives Considered

* Global install: depends on the machine, versions drift.
* Commit the collections: large third-party code in the repo.

## Related

* Extends: [ADR-0006](0006-taskfile-as-command-interface.md)
