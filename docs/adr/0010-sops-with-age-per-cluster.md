<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0010: SOPS with age, one key per cluster

## Status

Accepted

Date: 2026-10-25

## Context

App and cluster secrets should be in Git next to the manifests. Flux
decrypts SOPS files natively. A leaked key should expose only one
cluster. Re-keying files must still work if AWS or SSM is not available.

## Decision

* SOPS with age.
* Each cluster has its own age key. The private key is in SSM at
  `/projects/k8s-mgmt/<host>/age-private-key`, the public key at
  `.../age-public-key`.
* `.sops.yaml` has one rule per cluster folder. It encrypts `data` and
  `stringData` in `*.sops.yaml` files for the cluster key and a personal
  recovery key, kept outside AWS.
* The FluxInstance patches its Kustomization with SOPS decryption from
  the `sops-age` Secret.

## Consequences

* A file outside a cluster folder matches no rule, so `sops` refuses to
  encrypt it.
* Shared folders such as `infrastructure/` will need a rule with all
  cluster keys.
* The recovery key allows `sops updatekeys` and decryption without SSM.
* Each new cluster adds a key and a rule.

## Alternatives Considered

* One age key for all clusters: one leak exposes every cluster.
* No recovery key: re-keying depends on SSM being available.
* External Secrets Operator: see
  [ADR-0011](0011-age-key-delivery-with-task-secret.md).

## Related

* Extends: [ADR-0012](0012-one-inventory-host-per-cluster.md)
