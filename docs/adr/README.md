<!--
SPDX-FileCopyrightText: 2025 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# Architecture Decision Records

This directory holds the Architecture Decision Records (ADRs) for the
project.

## What are ADRs

An ADR is a short document that records one decision, why it was made,
and what follows from it. A reader new to the project, human or AI
agent, should understand the decision and its reasons from the ADR
alone, without reading the code. For the reasoning behind using them,
see
[ADR-0001: Use of Architecture Decision Records](0001-use-of-architecture-decision-records.md).

## Index of ADRs

- [ADR-0001: Use of Architecture Decision Records](0001-use-of-architecture-decision-records.md)
- [ADR-0002: k3s on Debian, installed once](0002-k3s-on-debian-installed-once.md)
- [ADR-0003: Separate bootstrap and site playbooks](0003-bootstrap-and-site-playbooks.md)
- [ADR-0004: Admin accounts per SSH key](0004-admin-accounts-per-ssh-key.md)
- [ADR-0005: Local Ansible collections](0005-local-ansible-collections.md)
- [ADR-0006: Taskfile as command interface](0006-taskfile-as-command-interface.md)
- [ADR-0007: Unattended OS updates](0007-os-updates.md)
- [ADR-0008: Cluster bootstrap on the node](0008-cluster-bootstrap-on-the-node.md)
- [ADR-0009: Flux Operator and FluxInstance bootstrap](0009-flux-operator-and-fluxinstance-bootstrap.md)
- [ADR-0010: SOPS with age, one key per cluster](0010-sops-with-age-per-cluster.md)
- [ADR-0011: Age key delivery over SSH, outside Ansible](0011-age-key-delivery-with-task-secret.md)
- [ADR-0012: One inventory host per cluster](0012-one-inventory-host-per-cluster.md)
- [ADR-0013: k3s upgrades with system-upgrade-controller](0013-k3s-upgrades-with-system-upgrade-controller.md)
- [ADR-0014: Nightly maintenance window](0014-nightly-maintenance-window.md)
- [ADR-0015: Site playbook in GitHub Actions](0015-site-playbook-in-github-actions.md)
- [ADR-0016: Dual-stack k3s](0016-dual-stack-k3s.md)
- [ADR-0017: cert-manager with Let's Encrypt](0017-cert-manager-with-lets-encrypt.md)
- [ADR-0018: system-upgrade-controller by release URL](0018-system-upgrade-controller-by-release-url.md)
- [ADR-0019: Dependency updates with Renovate](0019-dependency-updates-with-renovate.md)
- [ADR-0020: Public service account issuer](0020-public-service-account-issuer.md)
- [ADR-0021: AWS IAM roles for pods with amazon-eks-pod-identity-webhook](0021-aws-iam-roles-for-pods-with-pod-identity-webhook.md)
- [ADR-0022: Cluster hardening from the k3s CIS guide](0022-cluster-hardening-from-the-k3s-cis-guide.md)
- [ADR-0023: Sidecar-less mTLS with Istio ambient](0023-sidecar-less-mtls-with-istio-ambient.md)
- [ADR-0024: Network policies from one ResourceSet](0024-network-policies-from-one-resourceset.md)
- [ADR-0025: Kyverno for generated resources](0025-kyverno-for-generated-resources.md)
- [ADR-0026: App ingress with ListenerSets](0026-app-ingress-with-listenersets.md)

## When to write an ADR

Write an ADR when a choice affects how the project is built or run, and
someone could later ask why. Examples are the choice of a tool or
platform, a security or update policy, a rule for how components work
together, or the rejection of an obvious option. A small decision can be
worth an ADR too.

A bug fix, a refactoring or a config change within an existing decision
needs no ADR.

## What goes into an ADR

An ADR records:

- The problem, the constraints and the goal.
- The decision, stated as a rule or policy, for example "follow every
  release of the current major version".
- The chosen tools, platforms and approaches.
- The consequences, both good and bad.
- The options that were rejected, and why.

One decision per ADR. Several decisions that serve one goal get one ADR
each when each of them could be made without the others.

Write it for the project as a whole, not for one instance, host or
environment. Real names and examples belong in the project README.

## What stays out of an ADR

An ADR leaves out implementation details. They change with the code and
make the ADR go out of date. Examples:

- Versions, IP ranges, times, schedules and other config values.
- File paths, folder names and config file lines.
- Resource fields, labels, annotations, flags and variable names.
- Commands, task names and step-by-step procedures.
- Names of hosts, accounts and environments.

Names that are part of the decision are fine, such as a tool, a
protocol, a standard or an API kind.

A simple test: if a code change can make a sentence wrong while the
decision stays the same, the sentence belongs in a code comment or in
the project README, not in the ADR.

## Writing style

- Write for a reader who is new to the project.
- Use plain English, short sentences and active voice.
- Use full sentences. Bullets are fine for a list of rules or options,
  but each bullet is a full sentence.
- Keep an ADR to one or two pages.
- The title names the decision, for example "k3s upgrades with
  system-upgrade-controller", not "k3s upgrades".

## Status

- **Proposed**: under discussion, not in effect yet.
- **Accepted**: agreed and in effect.
- **Superseded by ADR-NNNN**: a newer ADR replaces the whole decision.
- **Deprecated**: no longer in effect, and nothing replaces it.

An ADR that a newer ADR extends or amends stays Accepted.

## Related ADRs

The Related section links an ADR to others with these terms:

- **Depends on**: this decision needs the other decision in place.
- **Extends**: this decision builds on the other one and changes none of
  it.
- **Amends**: this decision changes part of the other one. The rest
  stays in effect.
- **Supersedes**: this decision replaces the other one completely.

Extends, Amends and Supersedes each have a reverse link in the older
ADR: Extended by, Amended by and Superseded by.

## Changing an ADR

An accepted ADR keeps its decision. These edits are allowed:

- Fix wording, style and links.
- Remove implementation details.
- Update the status and the Related section.
- Replace a statement that a newer ADR made wrong with a link to that
  ADR.

A changed decision needs a new ADR that extends, amends or supersedes
the old one. Git keeps every earlier version.

## Creating a New ADR

1. Copy the [template](template.md) to a new file named
   `NNNN-title-with-hyphens.md`, with the next free number.
1. Fill in the sections. Follow the guidance in the template, then
   delete it.
1. Set the status to Proposed and the date to today.
1. Add a line for the ADR to the index above.
1. Add the reverse links to every ADR it extends, amends or supersedes.
   Set the status of a superseded ADR and fix statements in older ADRs
   that the new one makes wrong.
1. After review, set the status to Accepted.
