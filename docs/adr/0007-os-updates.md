<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0007: OS updates

## Status

Accepted

Date: 2026-10-24

## Context

OS updates should run without a person. A short daily downtime is
acceptable. A playbook rerun must not upgrade packages or reboot in the
middle of other work.

On Debian, `apt-daily-upgrade.timer` runs unattended-upgrade in a fixed
morning slot. unattended-upgrade calls `shutdown -r <time>`. systemd
moves a time that has already passed today to the next day.

## Decision

* `bootstrap.yml` runs a full upgrade and reboots if the system reports
  that a reboot is required.
* `site.yml` installs and enables unattended-upgrades, with an automatic
  reboot at a fixed time after the upgrade slot.
* The apt config files are validated with `apt-config` before they are
  written.

## Consequences

* A fresh node starts with current packages and kernel.
* Later updates install every morning, with a reboot after them when
  needed.
* `task site` never upgrades packages or reboots.

## Alternatives Considered

* Reboot time at the start of the upgrade slot: the upgrade runs later,
  so the reboot moves to the next day.
* Full upgrade in `site.yml`: a rerun could restart services or need a
  reboot.
* Change the Debian timers: more config to maintain for the same result.

## Related

* Depends on: [ADR-0003](0003-bootstrap-and-site-playbooks.md)
