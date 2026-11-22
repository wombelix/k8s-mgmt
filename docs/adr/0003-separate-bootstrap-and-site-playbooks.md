<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0003: Separate bootstrap and site playbooks

## Status

Accepted

Date: 2026-10-24

## Context

After a VM rebuild, only root can log in. The node hardening turns root
login off. One playbook that connects as root on the first run and as an
admin account later would fail halfway or lock itself out.

We want one entry point for all regular runs, not a manual order of
many runs. The playbooks must work for any host in the inventory,
without host names in the playbooks.

## Decision

We split the Ansible runs into a bootstrap playbook for the first run as
root and a site playbook for every later run.

- The bootstrap playbook connects as root. It creates the admin accounts
  from [ADR-0004](0004-admin-accounts-per-ssh-key.md), upgrades all
  packages and reboots if needed. It never changes the SSH server
  config.
- The site playbook connects as an admin account and is the single entry
  point for all other runs. It imports one playbook per area, such as the
  node and the cluster.
- The site playbook refuses to run as root, so it cannot turn off root
  login on a host without admin accounts.
- Both playbooks are idempotent. A rerun on a running node changes
  nothing.
- The playbooks run on every host in the inventory and take host values
  from it.

## Consequences

- After a rebuild, the bootstrap playbook runs once, then the site
  playbook. Later runs need only the site playbook. The README has the
  runbook.
- Once root login is off, the bootstrap playbook fails at connect and
  changes nothing. Its tasks can still run as an admin account.
- The site playbook cannot import the bootstrap playbook, because they
  connect as different users.

## Alternatives Considered

- One playbook, run as root first and as an admin account later, needs
  no second playbook. It fails halfway once root login is off.
- Detecting at runtime which user can connect gives one command for both
  cases. It is hard to follow and fragile.
- One playbook per area with a documented run order keeps each playbook
  small. It adds manual steps, and the order is easy to get wrong.

## Related

- Extended by: [ADR-0008](0008-cluster-bootstrap-on-the-node.md)
- Extended by: [ADR-0012](0012-one-inventory-host-per-cluster.md)
