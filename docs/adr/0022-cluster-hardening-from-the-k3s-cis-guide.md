<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0022: Cluster hardening from the k3s CIS guide

## Status

Accepted

Date: 2026-11-21

## Context

The clusters run on public VMs, and one operator runs all of them. The
k3s CIS hardening guide lists host, component and runtime controls. Some
of them only help in multi-tenant clusters, or when keys and data are
stored apart. Others add a way to fail without a security gain.

The goal is a smaller attack surface without new failure modes and
without manual work.

## Decision

We apply the controls from the k3s CIS guide that protect a
single-operator cluster, and skip the rest.

Applied:

- Pod Security Admission enforces `restricted` as the cluster default.
  kube-system is exempt. A namespace that needs more sets its own label.
- Every namespace has a default NetworkPolicy that allows ingress only
  from the same namespace and from the node. kube-system also allows the
  traffic the guide lists for it.
  [ADR-0025](0025-kyverno-for-generated-resources.md) sets how the
  policies are created.
- The API server writes an audit log of all writes at metadata level,
  rotated on the node.
- The guide's settings for the API server and the kubelet apply: pod
  tokens keep their real lifetime, each pod has a PID limit, idle
  streaming connections close, the kubelet accepts only modern TLS
  ciphers, and finished pods are deleted sooner.

Skipped:

- Secrets encryption at rest would put the encryption key on the same
  disk as the datastore.
- `protect-kernel-defaults` adds nothing, the kubelet already sets the
  same values. The flag only adds a way for the kubelet to fail on
  start.
- EventRateLimit needs an alpha API for its config and adds little for
  one operator.
- Stricter PKI file permissions add nothing, the affected files are
  public certificates.
- AlwaysPullImages is meant for multi-tenant clusters. Images with
  `latest` already get pulled on every pod start.
- Closing the API port is not possible. The operator has no static IP,
  and kubectl cannot start a tunnel from the kubeconfig.

## Consequences

- New workloads must meet `restricted`, or their namespace needs its own
  Pod Security label.
- Egress stays open.
- The k3s policy controller always allows traffic from the local node,
  so the API server reaches webhooks. With more than one node, webhook
  pods on another node would need a rule for the node addresses.
- The audit log stays on the node. Root on the VM can change it.
- The API port stays reachable from the internet. Client certificates
  and nightly k3s upgrades protect it.

## Alternatives Considered

- Full CIS compliance gives a clean report. The skipped controls add no
  protection here.
- A policy engine that creates NetworkPolicies for new namespaces was
  first rejected as one more component.
  [ADR-0025](0025-kyverno-for-generated-resources.md) added Kyverno for
  this later.
- A VPN or SSH tunnel for the API port would hide it from the internet.
  It is not usable from kubectl alone.

## Related

- Depends on: [ADR-0002](0002-k3s-on-debian-installed-once.md)
- Depends on: [ADR-0009](0009-flux-operator-and-fluxinstance-bootstrap.md)
- Extended by: [ADR-0023](0023-sidecar-less-mtls-with-istio-ambient.md)
- Amended by: [ADR-0024](0024-network-policies-from-one-resourceset.md)
- Amended by: [ADR-0025](0025-kyverno-for-generated-resources.md)
