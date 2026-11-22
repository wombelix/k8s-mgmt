<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0002: k3s on Debian, installed once

## Status

Accepted

Date: 2026-10-24

## Context

Each cluster is a single Hetzner Cloud VM. We need a small Kubernetes
distribution that needs little setup on the OS and can upgrade itself.
A playbook rerun must not change the running Kubernetes version. The
Kubernetes API must stay reachable if DNS fails.

## Decision

We run k3s on the latest stable Debian release and install it once.

- The OS is the latest stable Debian release.
- k3s comes from the official install script on the stable channel and
  is configured through its config file.
- Ansible runs the install script only when k3s is missing.
- system-upgrade-controller owns k3s upgrades, see
  [ADR-0013](0013-k3s-upgrades-with-system-upgrade-controller.md).
- The API server certificate covers the host name and the node's public
  IPv4 and IPv6 addresses.

## Consequences

- A playbook rerun never upgrades k3s. A change to the k3s config
  restarts k3s.
- Ansible and the upgrade controller never compete over the k3s
  version.
- Clients reach the API by IP address when DNS fails.
- Debian needs no extra OS setup for k3s.

## Alternatives Considered

- The RHEL family is a common server OS. It needs an extra kernel module
  package, firewalld and nm-cloud-setup turned off, and an SELinux policy
  for k3s.
- Talos Linux is an OS built only for Kubernetes. Unattended OS upgrades
  need talosctl or an extra controller.
- The k3s-ansible collection offers ready-made playbooks. It is
  third-party, and its version pin and upgrade playbook overlap with
  system-upgrade-controller.
- Running the install script on every run keeps k3s current without
  another tool. But k3s would upgrade on unrelated runs, and two tools
  would own the k3s version.

## Related

- Depends on: [ADR-0003](0003-separate-bootstrap-and-site-playbooks.md)
- Extended by: [ADR-0013](0013-k3s-upgrades-with-system-upgrade-controller.md)
- Amended by: [ADR-0016](0016-dual-stack-k3s.md)
