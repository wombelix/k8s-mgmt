<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0027: Shared Istio ingress Gateway in place of Traefik

## Status

Accepted

Date: 2026-11-22

## Context

Apps need an ingress on ports 80 and 443 over IPv4 and IPv6. k3s
bundles Traefik for this. With the mesh from
[ADR-0023](0023-sidecar-less-mtls-with-istio-ambient.md), the ingress
must reach the apps over mTLS. Istio excludes the namespace of the
bundled Traefik from the mesh by default, to protect system pods.

Port 80 must only serve ACME HTTP-01 challenges and redirect to HTTPS.
No app may serve plain HTTP.

The Gateway API is the Kubernetes standard API for ingress. k3s installs
its CRDs as a separate component, not as part of Traefik.

## Decision

The Istio ingress gateway with the Gateway API replaces the bundled
Traefik. One shared Gateway serves all apps.

- k3s runs without the bundled Traefik.
- One shared Gateway serves all apps, like name-based virtual hosts on a
  classic web server.
- The gateway is a mesh proxy itself and reaches the apps over mTLS. TLS
  from clients ends at the gateway.
- The gateway listens on both address families.
- On port 80, one route redirects every request to HTTPS.
  cert-manager's challenge routes name the exact host, so they take
  precedence over the redirect.
- cert-manager solves HTTP-01 through the Gateway with its Gateway API
  solver.
- [ADR-0026](0026-app-ingress-with-listenersets.md) sets how apps add
  their listeners, routes and certificates.

## Consequences

- Client traffic is encrypted up to the gateway, and from there to the
  app.
- The cluster serves no plain HTTP, except the short-lived challenge
  responses.
- Ingress and mesh come from one project and upgrade together.
- The Istio components reserve more memory than Traefik did. Their
  actual use is about the same.
- The Gateway API CRDs stay installed without Traefik.

## Alternatives Considered

- Keeping the bundled Traefik needs no change. Istio excludes its
  namespace from the mesh, so it could not reach the apps over mTLS.
- Traefik in its own namespace in the mesh keeps a known ingress. It is
  one more component, and Istio does not document it for ambient.
- One Gateway per app gives each app full control. Each Gateway adds its
  own gateway pods.

## Related

- Extends: [ADR-0023](0023-sidecar-less-mtls-with-istio-ambient.md)
- Amends: [ADR-0017](0017-cert-manager-with-lets-encrypt.md)
- Amended by: [ADR-0026](0026-app-ingress-with-listenersets.md)
