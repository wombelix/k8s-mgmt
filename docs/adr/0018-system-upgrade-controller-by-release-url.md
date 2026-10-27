<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0018: system-upgrade-controller by release URL

## Status

Accepted

Date: 2026-10-27

## Context

ADR-0013 copies the two release files of system-upgrade-controller into
the repo. A new release means downloading and replacing both files by
hand. Renovate can bump a version string, but the hosted Renovate App
cannot download files during an update.

An old controller can stop working with newer Kubernetes versions. Its
default config also pins the kubectl image for its Jobs, which only
moves with new controller releases.

Flux allows remote resources in Kustomizations by default.

## Decision

* The Kustomization in `infrastructure/system-upgrade/` references both
  release files by their release URL, with a pinned version.
* Renovate watches the upstream releases and opens a pull request that
  bumps the version in the URLs.

## Consequences

* New controller releases arrive as Renovate pull requests.
* The repo no longer holds the upstream files, and ansible-lint needs no
  excludes for them.
* Flux needs GitHub to build this folder. If GitHub is not reachable,
  the last applied state stays in place.
* GitHub release files can be replaced by the upstream maintainers. The
  URL does not pin the content.

## Alternatives Considered

* Keep the copied files: no automatic bump pull requests.
* Self-hosted Renovate with commands that download the files: needs
  hosting and a token for Renovate.
* Upstream repo through a GitRepository: its Kustomization has no CRD and
  pins an older controller image.

## Related

* Amends: [ADR-0013](0013-k3s-upgrades-with-system-upgrade-controller.md)
