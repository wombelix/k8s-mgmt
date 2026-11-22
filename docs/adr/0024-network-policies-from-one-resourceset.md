<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0024: Network policies from one ResourceSet

## Status

Superseded by [ADR-0025](0025-kyverno-for-generated-resources.md)

Date: 2026-11-22

## Context

[ADR-0022](0022-cluster-hardening-from-the-k3s-cis-guide.md) puts a
NetworkPolicy in every namespace, next to its resources. Every copy was
the same rule. Mesh namespaces now also need a rule for the ingress
gateway, see [ADR-0023](0023-sidecar-less-mtls-with-istio-ambient.md).
More copies mean more places to forget a change.

Kubernetes and the k3s policy controller have no cluster-wide
NetworkPolicy. Some object per namespace must exist.

## Decision

One Flux Operator ResourceSet renders the default ingress policy for
every namespace from one template and one list of namespaces.

- The default policy allows ingress only from the same namespace. The
  node is always allowed, so health checks and webhook calls work.
- A flag in the list marks mesh namespaces. They also allow the ingress
  gateway on the mesh port.
- Extra rules for one component stay next to that component.

## Consequences

- One file shows the default rule for every namespace.
- A new namespace needs one line in the list.
- The list is kept by hand, because the Flux Operator cannot list
  namespaces.

## Alternatives Considered

- A copy per namespace is the old way. Every change touches many files.
- A kustomize component per namespace avoids copies. It still needs one
  include per namespace, and the rules are spread over many folders.
- A policy engine such as Kyverno that creates policies for new
  namespaces needs no list. It is one more component with its own
  webhooks. It can replace the ResourceSet once other use cases need a
  policy engine.

## Related

- Superseded by: [ADR-0025](0025-kyverno-for-generated-resources.md)
- Amends: [ADR-0022](0022-cluster-hardening-from-the-k3s-cis-guide.md)
- Depends on: [ADR-0009](0009-flux-operator-and-fluxinstance-bootstrap.md)
