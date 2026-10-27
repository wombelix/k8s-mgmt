<!--
SPDX-FileCopyrightText: 2025 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# Architecture Decision Records

This directory holds the Architecture Decision Records (ADRs) for the
project.

## What are ADRs

An ADR is a short document that records one decision, why it was made,
and what follows from it. For the reasoning behind using them, see
[ADR-0001: Use of Architecture Decision Records](0001-use-of-architecture-decision-records.md).

## Index of ADRs

- [ADR-0001: Use of Architecture Decision Records](0001-use-of-architecture-decision-records.md)
- [ADR-0002: k3s on Debian, installed once](0002-k3s-on-debian-installed-once.md)
- [ADR-0003: Bootstrap and site playbooks](0003-bootstrap-and-site-playbooks.md)
- [ADR-0004: Admin accounts per SSH key](0004-admin-accounts-per-ssh-key.md)
- [ADR-0005: Local Ansible collections](0005-local-ansible-collections.md)
- [ADR-0006: Taskfile as command interface](0006-taskfile-as-command-interface.md)
- [ADR-0007: OS updates](0007-os-updates.md)
- [ADR-0008: Cluster bootstrap on the node](0008-cluster-bootstrap-on-the-node.md)
- [ADR-0009: Flux Operator and FluxInstance bootstrap](0009-flux-operator-and-fluxinstance-bootstrap.md)
- [ADR-0010: SOPS with age, one key per cluster](0010-sops-with-age-per-cluster.md)
- [ADR-0011: Age key delivery with task secret](0011-age-key-delivery-with-task-secret.md)
- [ADR-0012: One inventory host per cluster](0012-one-inventory-host-per-cluster.md)
- [ADR-0013: k3s upgrades with system-upgrade-controller](0013-k3s-upgrades-with-system-upgrade-controller.md)
- [ADR-0014: Nightly maintenance window](0014-nightly-maintenance-window.md)
- [ADR-0015: Site playbook in GitHub Actions](0015-site-playbook-in-github-actions.md)
- [ADR-0016: Dual-stack k3s](0016-dual-stack-k3s.md)
- [ADR-0017: cert-manager with Let's Encrypt](0017-cert-manager-with-lets-encrypt.md)
- [ADR-0018: system-upgrade-controller by release URL](0018-system-upgrade-controller-by-release-url.md)

## Writing an ADR

- One decision per ADR.
- Write it for the project as a whole, not for one instance, host or
  environment. Real names and examples belong in the README.
- Record the decision, its reasons, the chosen tools and approaches, and
  policies such as "latest stable release".
- Leave out implementation details: versions, IP ranges, times, exact
  config lines and commands. They change with the code. Name the file
  that holds them instead.
- Accepted ADRs are not edited. A new ADR supersedes or amends an old
  one.

## Creating a New ADR

1. Copy the [template](template.md) to a new file named `NNNN-title-with-hyphens.md`
1. Fill in the template sections
1. Add a line for the ADR to the index above
1. Submit for review and acceptance
