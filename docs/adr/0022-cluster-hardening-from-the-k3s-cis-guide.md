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

Applied:

* Pod Security Admission with `restricted` as the cluster default,
  through an admission config file. kube-system is exempt. A namespace
  that needs more sets its own label.
* NetworkPolicies in every namespace: ingress only from the same
  namespace. kube-system also allows DNS, metrics-server and the bundled
  ingress controller, as in the guide. The k3s policy controller always
  allows traffic from the local node, so on a single-node cluster the API
  server reaches webhooks.
* API server audit log at metadata level, without reads, Lease renewals
  and Events, rotated on the node.
* Pod tokens keep their real lifetime. A PID limit per pod. Idle
  streaming connections close. The kubelet accepts only modern TLS 1.2
  ciphers. Finished pods are deleted sooner.

Skipped:

* Secrets encryption at rest: the key is on the same disk as the
  datastore.
* `protect-kernel-defaults`: the kubelet already sets the same values.
  The flag only adds a way for the kubelet to fail on start.
* EventRateLimit: its config is an alpha API, little gain for one
  operator.
* PKI file permissions: the affected files are public certificates.
* AlwaysPullImages: meant for multi-tenant clusters. Images with
  `latest` already get pulled on every pod start.
* Closing the API port: no static IP on the operator side, and kubectl
  cannot start a tunnel from the kubeconfig.

## Consequences

* New workloads must meet `restricted`, or their namespace needs its own
  Pod Security label.
* Every new namespace needs its own NetworkPolicy. Egress stays open.
* With more than one node, the API server reaches webhook pods from
  another node. Namespaces with webhooks then need a rule that allows
  the node addresses.
* The audit log stays on the node. Root on the VM can change it.
* The API port stays reachable from the internet. Client certificates
  and nightly k3s upgrades protect it.
* The settings are in `ansible/node.yml` and next to each namespace in
  `infrastructure/`.

## Alternatives Considered

* Full CIS compliance: the skipped controls add no protection here.
* A policy engine that creates NetworkPolicies for new namespaces: one
  more component, not needed for a few namespaces.
* VPN or SSH tunnel for the API port: not usable from kubectl alone.

## Related

* Depends on: [ADR-0002](0002-k3s-on-debian-installed-once.md)
* Depends on: [ADR-0009](0009-flux-operator-and-fluxinstance-bootstrap.md)
