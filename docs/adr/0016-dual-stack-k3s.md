<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0016: Dual-stack k3s

## Status

Accepted

Date: 2026-10-27

## Context

Each node has an A and an AAAA record, and the cluster's ingress hosts
point to it. An IPv4-only k3s refuses IPv6 connections on the ingress
ports.

Let's Encrypt connects over IPv6 first when an AAAA record exists. It
retries over IPv4 only after a timeout, not after a refused connection.
HTTP-01 validation then fails for every ingress host.

k3s can only enable dual-stack when the cluster is first created.
Hetzner gives each VM one /64, which is too small for a pod range.

## Decision

k3s runs dual-stack, with IPv4 as the primary family.

- The IPv4 pod and service ranges stay the k3s defaults.
- The IPv6 pod and service ranges come from a random unique local
  prefix (RFC 4193).
- Flannel masquerades outgoing IPv6 traffic from pods.

## Consequences

- Ingress hosts answer over IPv4 and IPv6, and HTTP-01 works.
- ServiceLB serves both address families.
- Pods leave the node with the node's IPv6 address.
- An existing IPv4-only cluster needs a rebuild. A later change to the
  ranges needs one too.

## Alternatives Considered

- IPv4 only for web traffic needs no rebuild. The ingress hosts would
  point to a name without an AAAA record, and apps would have no IPv6.
- Public IPv6 for pods avoids NAT. The /64 per VM is too small for the
  pod range.
- IPv6 as the primary family is the more modern choice. It needs the
  node addresses set explicitly and gains nothing here.

## Related

- Amends: [ADR-0002](0002-k3s-on-debian-installed-once.md)
