<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0019: Dependency updates with Renovate

## Status

Accepted

Date: 2026-10-27

## Context

Flux follows version ranges for Flux itself, Flux Operator and the Helm
charts. Other dependencies are pinned in the repo: GitHub Actions,
pre-commit hooks, Ansible collections and the system-upgrade-controller
release. Nothing told us about new versions of those.

Automerge needs a check on pull requests, otherwise updates merge
without any test.

Renovate's ansible-galaxy manager cannot look up collections that are
installed from a git URL on GitHub.

## Decision

* The hosted Mend Renovate App runs on the repo, with the config in
  `renovate.json5`.
* Renovate updates GitHub Actions, pre-commit hooks, Ansible collections
  and, through a custom regex manager, the system-upgrade-controller
  release URLs.
* Minor, patch and digest updates merge automatically. Major updates
  wait for review.
* A pre-commit check runs on every pull request, on the changed files
  only. It uses a shared composite action from
  `wombelix/build-helper-scripts`.
* Ansible collections are installed from Ansible Galaxy, not from git.

## Consequences

* Pinned dependencies stay current without manual checks.
* An update only merges after the pre-commit check passes.
* `0.x` releases count minor bumps as minor, so they merge
  automatically too.
* Updates that Flux follows through version ranges are not part of
  Renovate.
* `task deps` downloads collections from Galaxy.

## Alternatives Considered

* Self-hosted Renovate: needs hosting and a token.
* Dependabot: no support for custom regex managers or Flux.
* Automerge without a check: updates merge without any test.
* Keep collections on git with a Renovate rule per collection: a
  workaround for each new collection.

## Related

* Amends: [ADR-0005](0005-local-ansible-collections.md)
* Depends on: [ADR-0018](0018-system-upgrade-controller-by-release-url.md)
