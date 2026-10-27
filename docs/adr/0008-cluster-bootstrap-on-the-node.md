<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0008: Cluster bootstrap on the node

## Status

Accepted

Date: 2026-10-24

## Context

The Flux bootstrap needs helm and kubectl with cluster admin access.
`site.yml` should run in a pipeline later. The k3s admin kubeconfig holds
a client certificate that works from anywhere on port 6443 and cannot be
revoked without a k3s certificate rotation. An SSH key can be removed
with one playbook run.

## Decision

* `ansible/cluster.yml` runs helm and `k3s kubectl` on the node with
  the node's admin kubeconfig.
* helm is installed once with the upstream `get-helm-4` script, which
  checks the sha256 sum. Debian has no helm package.
* `ansible/kubeconfig.yml` copies the kubeconfig to your machine for
  local tools. `site.yml` does not import it. The cluster, context and
  user use the host name.

## Consequences

* No admin kubeconfig leaves the node during a run, also not in CI.
* The node has helm installed, only used for the bootstrap.
* Day-to-day cluster changes go through Git, not SSH.

## Alternatives Considered

* Copy the kubeconfig to the controller and run helm there: a long-lived
  credential on runners, in caches and logs.
* Tooling only on the controller, port 6443 closed: every cluster task
  would need SSH or a tunnel.

## Related

* Extends: [ADR-0003](0003-bootstrap-and-site-playbooks.md)
