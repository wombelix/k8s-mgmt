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

Secrets in Git are encrypted with SOPS and age, with one age key per
cluster plus a recovery key.

- Each cluster has its own age key. The private and public key are in
  AWS SSM Parameter Store, one path per cluster.
- The SOPS config has one rule per cluster folder. It encrypts only the
  secret values, for the cluster key and for a personal recovery key that
  is kept outside AWS.
- Flux decrypts with a Secret that holds the cluster key, see
  [ADR-0011](0011-age-key-delivery-with-task-secret.md).

## Consequences

- A file outside a cluster folder matches no rule, so SOPS refuses to
  encrypt it.
- A shared folder outside the cluster folders would need a rule with all
  cluster keys.
- The recovery key allows re-keying and decryption without SSM.
- Each new cluster adds a key and a rule.
- Fields that are not secret stay readable in Git and in diffs.

## Alternatives Considered

- One age key for all clusters is one key to manage. One leak exposes
  every cluster.
- Without a recovery key there is one key less to protect. Re-keying
  then depends on SSM.
- External Secrets Operator is covered in
  [ADR-0011](0011-age-key-delivery-with-task-secret.md).

## Related

- Depends on: [ADR-0012](0012-one-inventory-host-per-cluster.md)
