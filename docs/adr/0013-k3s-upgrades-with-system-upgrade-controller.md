<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0013: k3s upgrades with system-upgrade-controller

## Status

Accepted

Date: 2026-10-26

## Context

Ansible installs k3s once and never upgrades it. k3s needs upgrades
without a person. The upgrade restarts k3s, so it must not run during the
day.

system-upgrade-controller is the upgrade method in the k3s docs. Its
release has two install files: `crd.yaml` and
`system-upgrade-controller.yaml`. The `kustomization.yaml` in the
upstream repo has no CRD and pins an older controller image than its own
release tag.

## Decision

* `infrastructure/system-upgrade/` holds both release files unmodified,
  from v0.20.2, plus a Plan.
* The Flux Kustomization `system-upgrade` in the cluster folder applies
  that folder.
* The Plan `k3s-server` follows the k3s docs example. It upgrades to the
  latest release in the stable channel, with `concurrency: 1` and
  `cordon: true`. There is no agent Plan, since each cluster has one
  node.
* The Plan has a window from 03:00 to 04:00 `Europe/Berlin`, every day.

## Consequences

* k3s follows the stable channel without a person.
* An upgrade Job only starts in the window. A Job that started may run
  past 04:00.
* The controller version stays at v0.20.2 until someone replaces the two
  files.
* ansible-lint skips the two upstream files.

## Alternatives Considered

* Upstream `kustomization.yaml` through a GitRepository: no CRD, and the
  image tag lags behind the release.
* Rancher Helm chart: hidden in the chart index and tied to Rancher
  versions.
* No window: k3s restarts within about 15 minutes of each new stable
  release, at any time of day.

## Related

* Extends: [ADR-0002](0002-k3s-on-debian-installed-once.md)
* Depends on: [ADR-0009](0009-flux-operator-and-fluxinstance-bootstrap.md)
