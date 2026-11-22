<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0025: Kyverno for generated resources

## Status

Accepted

Date: 2026-11-22

## Context

Every namespace needs a default NetworkPolicy, see
[ADR-0022](0022-cluster-hardening-from-the-k3s-cis-guide.md).
[ADR-0024](0024-network-policies-from-one-resourceset.md) renders it
from a ResourceSet with a list of namespaces, kept by hand in this repo.

Each app is meant to have its own repo and bring everything it
needs. A list in this repo means a change here for every new app.
Kubernetes has no cluster-wide NetworkPolicy, and the Flux Operator
cannot list namespaces. Creating an object for every new namespace is
the use case a policy engine with generate rules was built for.

## Decision

* Kyverno from the upstream OCI Helm chart, deployed by Flux. It
  follows every release of the current major chart version.
* Only the admission and background controllers run. The cleanup and
  reports controllers are off.
* One GeneratingPolicy creates the default ingress policy in every
  namespace, also in namespaces that already exist. Kyverno recreates it
  when someone changes or deletes it.
* The default policy allows ingress from the same namespace and from
  the ingress gateway on the mesh port. The node is always allowed, so
  health checks and webhook calls work. Namespaces outside the mesh have
  nothing listening on the mesh port.
* kube-system and Kyverno's own namespace keep their own policy files.
  Kyverno skips both.
* Validation rules that need no generated objects use the built-in
  ValidatingAdmissionPolicy instead, so they work without Kyverno.

## Consequences

* A new namespace gets its policy without any change in this repo.
* The webhook of a GeneratingPolicy never blocks a request. A namespace
  created while Kyverno is down has no policy until the namespace
  changes again.
* While Kyverno is down, objects it generated cannot be changed.
* Kyverno's namespace is excluded from the mesh, the API server calls
  its webhook from the node.
* Kyverno adds two pods and one more release that Flux upgrades.
* The settings are in `infrastructure/kyverno/`,
  `infrastructure/kyverno-policies/` and
  `infrastructure/network-policies/`.

## Alternatives Considered

* The ResourceSet from ADR-0024: a change in this repo for every app.
* A NetworkPolicy copy in every app repo: an app that forgets it is open
  to every pod in the cluster.
* Kyverno's older ClusterPolicy type: deprecated upstream.
* OPA Gatekeeper: validates and mutates, but cannot create objects.

## Related

* Supersedes: [ADR-0024](0024-network-policies-from-one-resourceset.md)
* Amends: [ADR-0022](0022-cluster-hardening-from-the-k3s-cis-guide.md)
* Depends on: [ADR-0023](0023-sidecar-less-mtls-with-istio-ambient.md)
