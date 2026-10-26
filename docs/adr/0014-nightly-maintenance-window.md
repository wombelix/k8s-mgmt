<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0014: Nightly maintenance window

## Status

Accepted

Date: 2026-10-26

## Context

The cluster must be available until at least 03:00 and again from 07:00,
in local time `Europe/Berlin`. k3s upgrades, OS upgrades and reboots must
all finish in between.

The node runs in UTC. The Debian apt timer starts at 06:00 UTC, and
ADR-0007 reboots at 07:30 UTC. That is 07:00 to 08:30 in winter and one
hour later in summer.

`Automatic-Reboot-Time` passes `HH:MM` to `shutdown -r`, which uses the
node's time zone. A fixed time moves by one hour at each DST change.
systemd timers accept an IANA time zone after the time and follow DST.

## Decision

All times in `Europe/Berlin`:

1. 03:00 to 04:00: k3s upgrade Jobs may start (ADR-0013).
2. 04:00 to 04:30: `apt-daily-upgrade.timer` starts unattended-upgrade.
   A drop-in sets `OnCalendar=*-*-* 04:00 Europe/Berlin` and
   `RandomizedDelaySec=30m`.
3. Right after the upgrade: reboot if needed, with
   `Automatic-Reboot-Time "now"`.

The node's time zone stays UTC.

## Consequences

* Maintenance usually ends before 05:00 and leaves two hours of buffer
  before 07:00.
* k3s and the OS upgrade one after the other, not at the same time.
* The schedule follows DST without changes.
* Log timestamps on the node stay in UTC.

## Alternatives Considered

* Node time zone `Europe/Berlin`: changes all log timestamps, and the
  timer still needs an override to start at 04:00.
* Fixed reboot time in UTC: moves by one hour at each DST change.
* OS upgrade before k3s: same result, but the k3s window would start
  earlier or end later.

## Related

* Amends: [ADR-0007](0007-os-updates.md)
* Depends on: [ADR-0013](0013-k3s-upgrades-with-system-upgrade-controller.md)
