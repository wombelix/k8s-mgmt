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

Each cluster signs ServiceAccount tokens for a public issuer URL, served
as static files by a Cloudflare Worker.

- Each cluster has its own issuer URL. All issuer URLs share one host
  and differ only in the path.
- The cluster signs tokens for the public issuer and still accepts
  tokens for the k3s default issuer, so existing tokens stay valid.
- A Cloudflare Worker with static assets serves the discovery document
  and the JWKS on a custom domain. The files and the Worker are in a
  separate repo for domain management.
- The signing key stays on the node. After a rebuild, the new JWKS is
  exported from the cluster and published.

## Consequences

- AWS roles trust one cluster through its issuer path.
- The private signing key never leaves the node.
- After a rebuild, pods get no AWS credentials until the new JWKS is
  published. Publishing is a manual step in a second repo.
- No bucket, no rewrite rule and no storage credentials in a pipeline
  are needed. Static asset requests are free.
- Bot protection on the zone stays off, because it could block the fetch
  from AWS STS.

## Alternatives Considered

- A backup of the signing key in SSM would keep the JWKS stable across
  rebuilds. A copy could sign tokens for every role that trusts the
  issuer.
- Serving the discovery endpoints from the API server needs no extra
  host. It needs anonymous access and a publicly trusted certificate on
  the API port.
- A Backblaze B2 or S3 bucket behind Cloudflare is a common setup. It
  needs a bucket, a rewrite rule for the paths and storage credentials
  in a pipeline.
- One issuer for all clusters is simpler. A trust policy for a
  ServiceAccount would match the same ServiceAccount in every cluster.

## Related

- Depends on: [ADR-0002](0002-k3s-on-debian-installed-once.md)
