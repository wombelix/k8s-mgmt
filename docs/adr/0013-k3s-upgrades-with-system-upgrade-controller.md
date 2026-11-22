<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0013: k3s upgrades with system-upgrade-controller

## Status

Accepted

Date: 2026-10-26

## Context

Ansible installs k3s once and never upgrades it, see
[ADR-0002](0002-k3s-on-debian-installed-once.md). k3s needs upgrades
without a person. An upgrade restarts k3s, so it must not run during
the day.

system-upgrade-controller is the upgrade method in the k3s docs. Its
release files are the complete install. The kustomization in the
upstream repo has no CRD and pins an older controller image than the
release.

## Decision

system-upgrade-controller upgrades k3s to the latest stable release,
only at night.

- Flux deploys the controller from its upstream release files,
  unmodified. [ADR-0018](0018-system-upgrade-controller-by-release-url.md)
  sets how Flux gets them.
- One Plan follows the k3s docs example for server nodes. It upgrades to
  the latest release in the stable channel. There is no Plan for agents,
  since each cluster has one node.
- The Plan has a daily time window, so upgrades start only at night. The
  window is set in [ADR-0014](0014-nightly-maintenance-window.md).

## Consequences

- k3s follows the stable channel without a person.
- An upgrade Job only starts in the window. A Job that started may run
  past its end.
- Each upgrade restarts k3s. On a single node, the API and new pod
  starts are unavailable for a short time.

## Alternatives Considered

- The upstream kustomization through a GitRepository needs only one
  reference to the upstream repo. It has no CRD, and its image lags
  behind the release.
- The Rancher Helm chart would deploy the controller like other Helm
  releases. It is hidden in the chart index and tied to Rancher
  versions.
- Without a window, upgrades arrive sooner. k3s would then restart
  shortly after each new stable release, at any time of day.

## Related

- Extends: [ADR-0002](0002-k3s-on-debian-installed-once.md)
- Depends on: [ADR-0009](0009-flux-operator-and-fluxinstance-bootstrap.md)
- Amended by: [ADR-0018](0018-system-upgrade-controller-by-release-url.md)
