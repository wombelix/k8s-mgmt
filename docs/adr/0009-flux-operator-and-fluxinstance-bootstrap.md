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

Ansible installs Flux Operator and the FluxInstance once, then Flux
takes over both.

- The cluster playbook installs Flux Operator with helm from the
  upstream OCI chart, only when the release is missing.
- A ResourceSet in the cluster folder takes over the operator release.
  It follows every new chart version.
- The cluster playbook applies the FluxInstance only when it is missing.
  Flux then syncs the same manifest from Git.
- The FluxInstance follows every release of the current Flux major
  version. It syncs the cluster folder from this public repo over HTTPS.
- Ansible uses the helm and kubectl CLIs, not the kubernetes.core
  collection.

## Consequences

- Flux and the operator upgrade without Ansible.
- Changes to the FluxInstance go through Git.
- No extra collection and no Python kubernetes library are needed.
- Flux needs no Git credentials, because the repo is public.
- A new Flux major version needs a change in Git.

## Alternatives Considered

- The kubernetes.core.helm module is the native Ansible way. It upgrades
  the release on every run when chart or values differ, which conflicts
  with Flux.
- The `flux-operator install` CLI does the install in one command.
  Upstream marks it for development and testing.
- `flux bootstrap github` is the classic Flux bootstrap. It needs a CLI
  run, and the operator already covers bootstrap and upgrades.
- A k3s HelmChart for the operator is built into k3s. The k3s
  helm-controller would keep owning the release next to Flux.

## Related

- Depends on: [ADR-0008](0008-cluster-bootstrap-on-the-node.md)
