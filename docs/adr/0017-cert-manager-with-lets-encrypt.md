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

The cluster has an ingress controller on ports 80 and 443 over IPv4 and
IPv6. Wildcard certificates need DNS-01 and API access to the DNS
provider.

Let's Encrypt limits failed validations and repeated certificates in
production. Its staging environment has much higher limits, but browsers
do not trust its certificates.

## Decision

* cert-manager from the upstream OCI Helm chart, deployed by Flux with
  an OCIRepository and a HelmRelease. It follows every release of the
  current major version.
* The chart installs and upgrades the CRDs. It keeps them when the
  release is removed.
* Two ClusterIssuers, Let's Encrypt staging and production. Both use
  HTTP-01 through the bundled ingress controller.
* A second Flux Kustomization applies the issuers. It depends on the
  cert-manager Kustomization, which waits until cert-manager is ready.
* Apps start with the staging issuer. They switch to production once
  they are ready for real use.

## Consequences

* Each host gets its own certificate. No DNS credentials in the cluster.
* Every host needs DNS that points to the node over all address
  families it has.
* Failed setups during development only use staging rate limits.
* An app needs one annotation change to go to production.
* The ACME account email is in the public repo.

## Alternatives Considered

* DNS-01 with wildcard certificates: needs DNS provider credentials in
  the cluster.
* Production issuer only: failed tests count against the production rate
  limits.
* Traefik's built-in ACME client: stores certificates in Traefik, not as
  Kubernetes Secrets, and ties TLS to one ingress controller.
* CRDs managed outside the chart: one more thing to upgrade by hand.

## Related

* Depends on: [ADR-0009](0009-flux-operator-and-fluxinstance-bootstrap.md)
* Depends on: [ADR-0016](0016-dual-stack-k3s.md)
