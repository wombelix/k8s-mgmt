<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0014: Nightly maintenance window

## Status

Accepted

Date: 2026-10-26

## Context

Clusters must be available from early morning until late at night, in
the operator's local time. k3s upgrades, OS upgrades and reboots must
all finish in a nightly window.

Nodes run in UTC. The Debian apt timer starts in a fixed UTC slot, and
ADR-0007 reboots at a fixed UTC time. Both fall into the morning in
local time.

`Automatic-Reboot-Time` passes `HH:MM` to `shutdown -r`, which uses the
node's time zone. A fixed time moves by one hour at each DST change.
systemd timers accept an IANA time zone after the time and follow DST.

## Decision

One nightly window in the operator's local time, in this order:

1. k3s upgrade Jobs may start (ADR-0013).
2. After that slot, a drop-in for `apt-daily-upgrade.timer` starts
   unattended-upgrade, with the time zone in the timer.
3. Right after the upgrade: reboot if needed.

Nodes keep UTC. The times and the time zone are in `ansible/node.yml`
and the k3s Plan.

## Consequences

* Maintenance usually ends well before the window closes.
* k3s and the OS upgrade one after the other, not at the same time.
* The schedule follows DST without changes.
* Log timestamps on the nodes stay in UTC.

## Alternatives Considered

* Node time zone set to local time: changes all log timestamps, and the
  timer still needs an override.
* Fixed reboot time in UTC: moves by one hour at each DST change.
* OS upgrade before k3s: same result, but the k3s slot would start
  earlier or end later.

## Related

* Amends: [ADR-0007](0007-os-updates.md)
* Depends on: [ADR-0013](0013-k3s-upgrades-with-system-upgrade-controller.md)
