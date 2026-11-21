<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0020: Public service account issuer

## Status

Accepted

Date: 2026-11-21

## Context

AWS STS and other OIDC consumers fetch the issuer's discovery document
and JWKS over HTTPS with a publicly trusted certificate. The k3s default
issuer is an in-cluster name. Its discovery endpoints on the API port
need authentication and use the k3s self-signed certificate.

k3s creates the token signing key on the first start. A VM rebuild
creates a new key. A copy of the private key could sign tokens for every
role that trusts the issuer.

## Decision

* Each cluster has its own issuer URL. All issuer URLs use the same
  host and differ only in the path, one path per cluster.
* The k3s config puts the public issuer first and keeps the k3s default
  issuer second, so tokens for the old issuer stay valid. The JWKS URI
  in the discovery document points to the public JWKS.
* The host is a Cloudflare Worker with static assets and a custom
  domain, configured in the domain management repo. The discovery
  document and the JWKS are files in that repo. The Worker serves them
  as `application/json`.
* The signing key stays on the node. After a rebuild, the new JWKS is
  exported from the cluster and published.

## Consequences

* AWS roles trust one cluster through its issuer path.
* The private key never leaves the node.
* After a rebuild, pods get no AWS credentials until the new JWKS is
  published. Publishing is a manual step in a second repo.
* No bucket, no rewrite rule and no storage credentials in a pipeline.
  Static asset requests are free.
* Bot protection on the zone stays off, because it could block the fetch
  from AWS STS.

## Alternatives Considered

* Back up the signing key to SSM and restore it after a rebuild: a copy
  could sign tokens for every role that trusts the issuer.
* Serve the discovery endpoints from the API server: needs anonymous
  access and a publicly trusted certificate on the API port.
* Backblaze B2 or S3 bucket behind Cloudflare: needs a bucket, a rewrite
  rule for the paths and storage credentials in a pipeline.
* One issuer for all clusters: a trust policy for a ServiceAccount would
  match the same ServiceAccount in every cluster.

## Related

* Depends on: [ADR-0002](0002-k3s-on-debian-installed-once.md)
