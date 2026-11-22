<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0018: system-upgrade-controller by release URL

## Status

Accepted

Date: 2026-10-27

## Context

[ADR-0013](0013-k3s-upgrades-with-system-upgrade-controller.md) first
copied the two release files of system-upgrade-controller into the repo.
A new release meant downloading and replacing both files by hand.
Renovate can bump a version string, but the hosted Renovate App cannot
download files during an update.

An old controller can stop working with newer Kubernetes versions. Its
default config also pins the kubectl image for its Jobs, which only
moves with new controller releases.

Flux allows remote resources in Kustomizations by default.

## Decision

Flux builds system-upgrade-controller from the release files at their
upstream URL, with a pinned version.

- A kustomization references both release files by their release URL.
- Renovate watches the upstream releases and opens a pull request that
  bumps the version in the URLs.

## Consequences

- New controller releases arrive as Renovate pull requests.
- The repo no longer holds copies of upstream files.
- Flux needs GitHub to build this folder. If GitHub is not reachable,
  the last applied state stays in place.
- The upstream maintainers can replace GitHub release files. The URL
  does not pin the content.

## Alternatives Considered

- Keeping the copied files works without GitHub at build time. There are
  no automatic bump pull requests.
- Self-hosted Renovate could download the files in an update. It needs
  hosting and a token.
- The upstream repo through a GitRepository needs no URL per file. Its
  kustomization has no CRD and pins an older controller image.

## Related

- Amends: [ADR-0013](0013-k3s-upgrades-with-system-upgrade-controller.md)
