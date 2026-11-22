<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0023: Sidecar-less mTLS with Istio ambient

## Status

Accepted

Date: 2026-11-22

## Context

Traffic between pods is plain text on the node's pod network. Another
pod or a process with network access on the node can read it, and it may
hold secrets. The goal is encryption in transit for all app traffic,
also between pods on the same node, and also from the ingress to the
apps.

The cluster keeps its bundled CNI. Pods must still pass Pod Security
`restricted`. The solution must update itself like the rest of the
cluster and need no work per app.

Infrastructure components already use TLS for all their traffic, for
example to the API server, to Let's Encrypt and for webhook calls. The
API server calls webhooks from the node, outside any mesh.

## Decision

Istio in ambient mode encrypts all app traffic between pods with mTLS,
without sidecars.

- Flux deploys Istio ambient from the upstream OCI Helm charts, in the
  order the Istio docs give. It follows every release of the current
  major version.
- A node agent, ztunnel, handles mTLS for every pod in the mesh. No pod
  gets a sidecar.
- The Istio CNI agent chains after the bundled CNI and only redirects
  traffic of pods in the mesh.
- Enrollment is opt-out. Every pod joins the mesh, except pods in a list
  of excluded infrastructure namespaces and pods with the opt-out label.
- One mesh-wide policy sets mTLS to `STRICT`. Pods in the mesh refuse
  plain text from outside the mesh.
- Istio runs dual-stack, like the cluster.

## Consequences

- App traffic between pods is encrypted with mTLS and has a workload
  identity. Traffic on the same node is encrypted too.
- Apps need no sidecar, no label and no change to pass Pod Security.
- A new infrastructure namespace must be added to the exclude list, or
  its pods join the mesh.
- A mesh namespace needs a network rule that lets the ingress gateway
  in. [ADR-0025](0025-kyverno-for-generated-resources.md) adds it to
  every namespace.
- Pods in the mesh lose their network while ztunnel restarts, for
  example during an Istio upgrade. The outage is short.
- A pod outside the mesh can still send a request to a mesh pod before
  `STRICT` rejects the connection. It never gets an answer.
- Istio adds a control plane and two node agents.
- The Istio namespace runs at Pod Security `privileged`, because the CNI
  agent needs host paths.

## Alternatives Considered

- Istio with sidecars is the older, widely used mode. It adds one
  container per pod and more resources, and the injection conflicts
  with Pod Security `restricted`.
- A CNI with encryption, such as Cilium or Calico with WireGuard,
  encrypts at the network layer. It replaces the bundled CNI and needs a
  cluster rebuild.
- TLS in every app needs no mesh. It is work per app, and easy to
  forget.
- Opt-in enrollment per namespace label is explicit. It is work per app,
  and a missing label leaves traffic in plain text.
- Enrolling the infrastructure namespaces would cover all traffic. Their
  traffic already uses TLS. A broken mesh would block Flux, k3s upgrades
  and webhooks, and `STRICT` would refuse the API server's webhook
  calls.

## Related

- Depends on: [ADR-0009](0009-flux-operator-and-fluxinstance-bootstrap.md)
- Depends on: [ADR-0016](0016-dual-stack-k3s.md)
- Extends: [ADR-0022](0022-cluster-hardening-from-the-k3s-cis-guide.md)
- Extended by: [ADR-0027](0027-shared-istio-ingress-gateway-in-place-of-traefik.md)
