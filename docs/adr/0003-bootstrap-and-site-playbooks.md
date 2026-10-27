<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0003: Bootstrap and site playbooks

## Status

Accepted

Date: 2026-10-24

## Context

After a VM rebuild only root can log in. Our hardening turns root login
off. One playbook that connects as root on the first run and as an admin
user later would fail halfway or lock itself out. We also want a single
entry point, not a manual order of many runs.

## Decision

* `ansible/bootstrap.yml` connects as root, set as a play var. It only
  creates the admin users, upgrades all packages and reboots if needed.
  It never touches sshd.
* `ansible/site.yml` connects as the inventory user. It imports
  `node.yml` and `cluster.yml` with `import_playbook`.
* `node.yml` refuses to run as root, so it cannot disable root login on
  a host without admin users.
* Both playbooks are idempotent. A rerun on a running node reports
  `changed=0`.
* Plays use `hosts: all`. Host values come from the inventory and
  `inventory_hostname`.

## Consequences

* After a rebuild: `task bootstrap`, then `task site`. Later only
  `task site`.
* Once root login is off, `task bootstrap` fails at connect and changes
  nothing. Its tasks can still run as an admin user.
* `bootstrap.yml` cannot be imported into `site.yml`.

## Alternatives Considered

* One playbook run once as root and later as an admin user: fails
  halfway once root login is off.
* Detect at runtime which user can connect: hard to follow and fragile.
* One playbook per concern with a documented run order: more manual
  steps.

## Related

* Extends: [ADR-0004](0004-admin-accounts-per-ssh-key.md)
