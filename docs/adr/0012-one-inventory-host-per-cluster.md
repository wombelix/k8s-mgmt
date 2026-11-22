<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0012: One inventory host per cluster

## Status

Accepted

Date: 2026-10-25

## Context

The repo should manage more than one cluster. Adding a cluster should
need no change to the playbooks or the Taskfile.

## Decision

Each host in the Ansible inventory is one single-node cluster.

- Settings that all clusters share are inventory variables for all
  hosts.
- Each cluster has its own folder in the repo. The folder name is
  derived from the host name, so the playbooks find it without extra
  config.
- Each cluster folder has its own FluxInstance that syncs only that
  folder.
- A run can be limited to one cluster with the Ansible host limit.

## Consequences

- A new cluster needs an inventory entry, DNS, an age key, a SOPS rule
  and a cluster folder. The README has the runbook.
- Multi-node clusters with k3s agents are not supported.
- All clusters share the playbooks and the infrastructure manifests, so
  a change there reaches every cluster.

## Alternatives Considered

- One repo per cluster gives full separation. The shared playbooks and
  infrastructure would be copied.
- Inventory groups per cluster with several nodes would allow multi-node
  clusters. They are not needed for single-node clusters.

## Related

- Extends: [ADR-0003](0003-separate-bootstrap-and-site-playbooks.md)
