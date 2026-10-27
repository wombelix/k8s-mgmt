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

* `infrastructure/system-upgrade/` holds both files of one release
  unmodified, plus a Plan.
* A Flux Kustomization in the cluster folder applies that folder.
* The Plan follows the k3s docs example for server nodes and upgrades to
  the latest release in the stable channel. There is no agent Plan,
  since each cluster has one node.
* The Plan has a daily time window, so upgrades only start at night.

## Consequences

* k3s follows the stable channel without a person.
* An upgrade Job only starts in the window. A Job that started may run
  past its end.
* The controller version stays the same until someone replaces the two
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
