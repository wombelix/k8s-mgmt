<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0009: Flux Operator and FluxInstance bootstrap

## Status

Accepted

Date: 2026-10-24

## Context

Flux should manage the cluster and upgrade itself. Upstream automates
Flux Operator upgrades only if the operator was installed with Helm.
After the bootstrap, Flux owns the operator release and the FluxInstance.
Any tool that keeps changing them would conflict with Flux.

## Decision

* `cluster.yml` installs Flux Operator with helm from the upstream OCI
  chart, only when the release is missing.
* A ResourceSet in the cluster folder takes over the release with an
  OCIRepository and a HelmRelease that follow every new chart version.
* `cluster.yml` applies the FluxInstance only when it is missing. Flux
  then syncs the same file from Git.
* The FluxInstance follows every release of the current Flux major
  version and syncs the cluster folder from this public repo over HTTPS.
* Ansible uses the helm and kubectl CLIs, not `kubernetes.core`.

## Consequences

* Flux and the operator upgrade without Ansible.
* Changes to the FluxInstance go through Git.
* No extra collection and no Python `kubernetes` library.

## Alternatives Considered

* `kubernetes.core.helm`: upgrades the release on every run when chart
  or values differ, which conflicts with Flux.
* `flux-operator install` CLI: upstream marks it for development and
  testing.
* `flux bootstrap github`: needs a CLI run, the operator covers
  bootstrap and upgrades.
* k3s HelmChart for the operator: the k3s helm-controller would keep
  owning the release next to Flux.

## Related

* Depends on: [ADR-0008](0008-cluster-bootstrap-on-the-node.md)
