<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0007: Unattended OS updates

## Status

Accepted

Date: 2026-10-24

## Context

OS updates should run without a person. A short reboot each night is
acceptable. A playbook rerun must not upgrade packages or reboot in the
middle of other work. A fresh node should start with current packages.

## Decision

Debian installs updates on its own with unattended-upgrades. The
playbooks upgrade packages only at bootstrap.

- The bootstrap playbook runs a full upgrade and reboots if the system
  needs it.
- The site playbook installs and enables unattended-upgrades. It
  installs all updates for the current Debian release and reboots after
  them if needed. The time is set in
  [ADR-0014](0014-nightly-maintenance-window.md).
- Major Debian releases stay manual, for example with a VM rebuild.
- The apt config is validated before it is written.

## Consequences

- A fresh node starts with current packages and kernel.
- Later updates install every night, with a reboot when needed.
- The site playbook never upgrades packages or reboots.
- A new major Debian release needs manual work.

## Alternatives Considered

- A full upgrade in the site playbook would update the node with every
  run. A rerun could then restart services or need a reboot.

## Related

- Depends on: [ADR-0003](0003-bootstrap-and-site-playbooks.md)
- Amended by: [ADR-0014](0014-nightly-maintenance-window.md)
