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
The Kubernetes API must stay reachable if DNS fails.

## Decision

* The latest stable Debian release as the OS.
* k3s from the official install script on the stable channel,
  configured with the k3s config file.
* Ansible runs the script only when k3s is missing.
* system-upgrade-controller will own k3s upgrades.
* The API server certificate covers the host name and the node's public
  IPv4 and IPv6 addresses, taken from the Ansible facts.

## Consequences

* A rerun of the playbook never upgrades or restarts k3s.
* Until system-upgrade-controller is set up, k3s is not upgraded.
* A change to the k3s config file restarts k3s through a handler.

## Alternatives Considered

* RHEL family: needs kernel-modules-extra, firewalld off, nm-cloud-setup
  off and an SELinux policy for k3s.
* Talos Linux: unattended OS upgrades need talosctl or an extra
  controller.
* k3s-ansible collection: third-party, and its version pin and upgrade
  playbook overlap with system-upgrade-controller.
* Rerun the install script on every run: k3s would upgrade on unrelated
  runs, and two tools would own the k3s version.

## Related

* Depends on: [ADR-0003](0003-bootstrap-and-site-playbooks.md)
