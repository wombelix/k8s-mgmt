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

On Debian 13, `apt-daily-upgrade.timer` runs unattended-upgrade between
06:00 and 07:00. unattended-upgrade calls `shutdown -r <time>`. systemd
moves a time that has already passed today to the next day.

## Decision

* `bootstrap.yml` runs a full upgrade (`apt-get dist-upgrade`) and
  reboots if `/run/reboot-required` exists.
* `site.yml` installs unattended-upgrades, writes `20auto-upgrades` and
  sets `Automatic-Reboot "true"` with `Automatic-Reboot-Time "07:30"`.
* Both config files are validated with `apt-config` before they are
  written.

## Consequences

* A fresh node starts with current packages and kernel.
* Later updates install every morning, with a reboot at 07:30 when
  needed.
* `task site` never upgrades packages or reboots.

## Alternatives Considered

* Reboot time 06:00: the upgrade runs after 06:00, so the reboot moves
  to the next day.
* Full upgrade in `site.yml`: a rerun could restart services or need a
  reboot.
* Change the Debian timers: more config to maintain for the same result.

## Related

* Depends on: [ADR-0003](0003-bootstrap-and-site-playbooks.md)
