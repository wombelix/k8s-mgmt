<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0008: Cluster bootstrap on the node

## Status

Accepted

Date: 2026-10-24

## Context

The Flux bootstrap needs helm and kubectl with cluster admin access. The
site playbook also runs in a pipeline. The k3s admin kubeconfig holds a
client certificate that works from anywhere on the API port. It cannot
be revoked without a k3s certificate rotation. An SSH key can be removed
with one playbook run.

## Decision

The cluster bootstrap runs helm and kubectl on the node, so the admin
kubeconfig never leaves the node during a run.

- The cluster playbook runs helm and the kubectl bundled with k3s on the
  node, with the node's admin kubeconfig.
- helm is installed once from the upstream install script, which checks
  the checksum of the download. Debian has no helm package.
- A separate playbook copies the kubeconfig to the operator's machine
  for local tools. The site playbook does not import it.

## Consequences

- No admin kubeconfig leaves the node during a run, also not in CI.
- The node has helm installed, used only for the bootstrap.
- Day-to-day cluster changes go through Git, not SSH.
- The local copy of the kubeconfig has the same certificate. It must
  never go to CI.

## Alternatives Considered

- Copying the kubeconfig to the controller and running helm there needs
  no tools on the node. It leaves a long-lived credential on runners, in
  caches and in logs.
- Keeping all tools on the controller with the API port closed hides the
  API from the internet. Every cluster task would then need SSH or a
  tunnel.

## Related

- Extends: [ADR-0003](0003-separate-bootstrap-and-site-playbooks.md)
