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
apps. Port 80 must only serve ACME HTTP-01 challenges and redirect to
HTTPS. No app may serve plain HTTP.

The cluster keeps its bundled CNI. Pods must still pass Pod Security
`restricted`. The solution must update itself like the rest of the
cluster and need no work per app.

Infrastructure components already use TLS for all their traffic, for
example to the API server, to Let's Encrypt and for webhook calls. The
API server calls webhooks from the node, outside any mesh.

## Decision

* Istio in ambient mode from the upstream OCI Helm charts, deployed by
  Flux in the documented order. It follows every release of the current
  major version. Ambient adds no sidecar. A node agent, ztunnel,
  handles mTLS for every pod in the mesh.
* The Istio CNI agent chains after the bundled CNI and only redirects
  traffic of pods in the mesh.
* Opt-out enrollment: every pod joins the mesh, except pods in a list of
  excluded infrastructure namespaces and pods labeled
  `istio.io/dataplane-mode=none`.
* One mesh-wide PeerAuthentication with `STRICT`. Pods in the mesh
  refuse plain text from outside the mesh.
* Istio is dual-stack, so the ingress gateway listens on both address
  families.
* The Istio ingress gateway with the Gateway API replaces the bundled
  Traefik. One shared Gateway serves all apps, like name-based virtual
  hosts on a classic web server.
* The Gateway's port 80 listener only accepts routes from its own
  namespace. One route there redirects every request to HTTPS.
  cert-manager's challenge routes name the exact host, so they take
  precedence over the redirect.
* Certificates for the shared Gateway belong in the Gateway's namespace.
  cert-manager solves HTTP-01 through the Gateway with the Gateway API
  solver.

## Consequences

* App traffic between pods is encrypted with mTLS and has a workload
  identity. Same-node traffic is encrypted too.
* Apps need no sidecar, no label and no change to pass Pod Security.
* The ingress gateway is part of the mesh and reaches the apps over
  mTLS. TLS from clients ends at the gateway.
* A new infrastructure namespace must be added to the exclude list, or
  its pods join the mesh. A mesh namespace needs a network rule that lets
  the gateway in, see
  [ADR-0024](0024-network-policies-from-one-resourceset.md).
* Pods in the mesh lose their network while ztunnel restarts, for
  example during an Istio upgrade. The outage is short.
* A pod outside the mesh can still send a request to a mesh pod before
  STRICT rejects the connection. It never gets an answer.
* Istio adds a control plane and two node agents. Their requests reserve
  more memory than Traefik did, their use is about the same.
* The Istio namespace runs at Pod Security `privileged`, because the CNI
  agent needs host paths.
* Disabling Traefik keeps the Gateway API CRDs, k3s installs them as a
  separate component.
* The settings are in `infrastructure/istio/`,
  `infrastructure/istio-config/`, `infrastructure/istio-ingress/`,
  `infrastructure/cert-manager-issuers/` and `ansible/node.yml`.

## Alternatives Considered

* Istio with sidecars: one extra container per pod, more resources, and
  injection that fights Pod Security `restricted`.
* A CNI with encryption such as Cilium or Calico with WireGuard: replaces
  the bundled CNI and needs a cluster rebuild.
* TLS in every app: work per app, and easy to forget.
* Opt-in enrollment per namespace label: work per app, a missing label
  leaves traffic in plain text.
* Enrolling infrastructure namespaces: their traffic already uses TLS.
  A broken mesh would block Flux, k3s upgrades and webhooks, and STRICT
  would refuse the API server's webhook calls.
* Keeping Traefik in its own namespace in the mesh: one more component,
  and Istio does not document it for ambient.
* Keeping the bundled Traefik in the mesh: Istio excludes its namespace
  by default to protect system pods.

## Related

* Depends on: [ADR-0009](0009-flux-operator-and-fluxinstance-bootstrap.md)
* Depends on: [ADR-0016](0016-dual-stack-k3s.md)
* Amends: [ADR-0017](0017-cert-manager-with-lets-encrypt.md)
* Extends: [ADR-0022](0022-cluster-hardening-from-the-k3s-cis-guide.md)
