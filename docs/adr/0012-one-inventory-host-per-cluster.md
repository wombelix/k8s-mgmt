<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0012: One inventory host per cluster

## Status

Accepted

Date: 2026-10-25

## Context

The repo should manage more than one cluster later. Adding a cluster
should not need changes to the playbooks or the Taskfile.

## Decision

* Each host in `ansible/inventory.yml` is one single-node cluster.
  Shared settings are in `all.vars`.
* The cluster folder is the host name with dashes instead of dots, for
  example `clusters/k8s-example-com/` for `k8s.example.com`.
  `cluster.yml` derives it from `inventory_hostname`.
* Each cluster folder has its own `flux-instance.yaml` with its own
  `sync.path`.
* `--limit <host>` runs a task for one cluster.

## Consequences

* A new cluster needs an inventory line, DNS, an age key, a `.sops.yaml`
  rule and a cluster folder. The README has the runbook.
* Multi-node clusters with k3s agents are not supported.

## Alternatives Considered

* One repo per cluster: shared playbooks and infrastructure would be
  copied.
* Inventory groups per cluster with several nodes: not needed for
  single-node clusters.

## Related

* Extends: [ADR-0003](0003-bootstrap-and-site-playbooks.md)
