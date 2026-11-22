<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0017: cert-manager with Let's Encrypt

## Status

Accepted

Date: 2026-10-27

## Context

Apps need trusted TLS certificates. Some domains, for example `.dev`, are
on the HSTS preload list, so browsers require HTTPS for every host.

The cluster has an ingress on ports 80 and 443 over IPv4 and IPv6.
Wildcard certificates need DNS-01 and API access to the DNS provider.

Let's Encrypt limits failed validations and repeated certificates in
production. Its staging environment has much higher limits, but browsers
do not trust its certificates.

## Decision

cert-manager issues a certificate per host from Let's Encrypt with
HTTP-01.

- Flux deploys cert-manager from the upstream OCI Helm chart. It follows
  every release of the current major version.
- The chart installs and upgrades the CRDs. It keeps them when the
  release is removed.
- Two ClusterIssuers point to Let's Encrypt staging and production. Both
  solve HTTP-01 through the cluster ingress.
  [ADR-0027](0027-shared-istio-ingress-gateway-in-place-of-traefik.md)
  moved this to the Istio Gateway.
- Flux applies the issuers only after cert-manager is ready.
- Apps start with the staging issuer. They switch to production once
  they are ready for real use.

## Consequences

- Each host gets its own certificate. No DNS credentials are in the
  cluster.
- Every host needs DNS that points to the node over all address families
  it has.
- Failed setups during development only use staging rate limits.
- An app switches to production by changing the issuer of its
  certificate.
- Removing cert-manager keeps the CRDs, so existing certificates stay.
- The ACME account email is in the public repo.

## Alternatives Considered

- DNS-01 with wildcard certificates gives one certificate for all hosts.
  It needs DNS provider credentials in the cluster.
- A production issuer only is simpler. Failed tests count against the
  production rate limits.
- The built-in ACME client of an ingress controller, such as Traefik,
  needs no extra component. It stores certificates in the controller,
  not as Kubernetes Secrets, and ties TLS to one ingress controller.
- CRDs managed outside the chart can be upgraded on their own schedule.
  They are one more thing to upgrade by hand.

## Related

- Depends on: [ADR-0009](0009-flux-operator-and-fluxinstance-bootstrap.md)
- Depends on: [ADR-0016](0016-dual-stack-k3s.md)
- Amended by: [ADR-0027](0027-shared-istio-ingress-gateway-in-place-of-traefik.md)
