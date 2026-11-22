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

Nodes run in UTC. Debian's default upgrade timer falls into the morning
in local time. unattended-upgrades schedules its reboot for a fixed time
of day in the node's time zone. That time moves by one hour at each DST
change. If the time has already passed when the upgrade ends, the reboot
moves to the next day. systemd timers accept an IANA time zone and
follow DST.

## Decision

All maintenance runs in one nightly window in the operator's local
time, in a fixed order:

1. k3s upgrade Jobs may start, see
   [ADR-0013](0013-k3s-upgrades-with-system-upgrade-controller.md).
2. After that slot, the Debian upgrade timer starts unattended-upgrades.
   The timer uses the operator's time zone, so it follows DST.
3. Right after the upgrade, the node reboots if needed.

Nodes keep UTC.

## Consequences

- Maintenance usually ends well before the window closes.
- k3s and the OS upgrade one after the other, not at the same time.
- The schedule follows DST without changes.
- Log timestamps on the nodes stay in UTC.
- The Debian upgrade timer has a local override that must be kept.
- The project README lists the times.

## Alternatives Considered

- Setting the node time zone to local time would put everything in one
  time zone. It changes all log timestamps, and the timer still needs an
  override.
- A fixed reboot time is the default way of unattended-upgrades. It
  moves by one hour at each DST change.
- A reboot time at the start of the upgrade slot keeps one fixed time.
  The upgrade runs later, so the reboot moves to the next day.
- The OS upgrade before k3s gives the same result. The k3s slot would
  start earlier or end later.

## Related

- Amends: [ADR-0007](0007-unattended-os-updates.md)
- Depends on: [ADR-0013](0013-k3s-upgrades-with-system-upgrade-controller.md)
