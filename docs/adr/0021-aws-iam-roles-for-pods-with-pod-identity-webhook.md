<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0021: AWS IAM roles for pods with amazon-eks-pod-identity-webhook

## Status

Accepted

Date: 2026-11-21

## Context

Apps call AWS services, without long-lived access keys. With the public
issuer from ADR-0020, a pod can exchange a projected ServiceAccount
token for temporary credentials. Each pod then needs a token volume with
the STS audience, a mount and environment variables.

Charts with IRSA support expect the EKS ServiceAccount annotation for
the role ARN.

## Decision

* amazon-eks-pod-identity-webhook, deployed by Flux. Upstream ships only
  raw manifests. The Kustomization references them by release tag and
  patches the namespace and certificate names. Renovate bumps the
  manifests and the image together.
* cert-manager issues the webhook's serving certificate from a
  self-signed Issuer in the webhook's namespace and injects the CA.
* Apps annotate their ServiceAccount with the role ARN. At pod creation
  the webhook adds the token volume, the mount, the environment
  variables and a default region.
* The IAM OIDC provider and the roles are created in an AWS account
  outside this repo.

## Consequences

* One annotation per app. Charts with IRSA support work unchanged.
* Tools that read the annotation themselves, for example External
  Secrets Operator, work without the webhook.
* The webhook mutates pods only at creation. After an annotation change,
  the pods need a restart.
* The webhook ignores its own failures. Pods created while it is down
  start without credentials and need a restart.
* Flux needs GitHub to build the folder, as in ADR-0018.

## Alternatives Considered

* Token volume, mount and environment variables in every pod: repeats
  the same block everywhere. Charts need support for extra volumes and
  environment variables.
* MutatingAdmissionPolicy: CEL cannot read the ServiceAccount, so the
  role ARN would go into a pod annotation or a ConfigMap parameter.
  Own CEL to write and test, and no IRSA chart support.
* Access keys in Secrets: long-lived credentials.

## Related

* Depends on: [ADR-0020](0020-public-service-account-issuer.md)
* Depends on: [ADR-0017](0017-cert-manager-with-lets-encrypt.md)
* Depends on: [ADR-0019](0019-dependency-updates-with-renovate.md)
