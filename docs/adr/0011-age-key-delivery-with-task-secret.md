<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0011: Age key delivery with task secret

## Status

Accepted

Date: 2026-10-25

## Context

Flux needs the cluster's age private key as a Secret. It is the only
secret that is not in Git. It must get from SSM to the cluster
without being written to disk or exposed to other processes.

Through Ansible, the age key would be in an environment variable, in play
and registered variables, and in the module file Ansible writes to the
node.

## Decision

`task secret`:

1. Reads the age key with the AWS CLI into an unexported shell variable.
   It stops if the read fails or returns nothing.
2. Pipes it with a shell builtin over SSH into `k3s kubectl` on the
   node, which creates or updates the Secret from stdin.

It connects with the account for manual runs by default. The pipeline
can pass its own account.

## Consequences

* The age key moves only through pipes. No file, no environment variable, no
  process argument, no Ansible.
* It runs after a rebuild and after a key rotation, not on every
  `task site`.
* The machine that runs it needs the AWS CLI and an AWS login.

## Alternatives Considered

* Ansible tasks with the age key from an environment variable: leaks as
  described above.
* params2env: prints `export KEY="value"`, the raw value needs text
  parsing.
* External Secrets Operator with static IAM keys: a long-lived AWS
  credential becomes the bootstrap secret.
* External Secrets Operator with OIDC federation: no bootstrap secret,
  but needs a public OIDC issuer and a JWKS update after every rebuild.
  Possible later next to SOPS.

## Related

* Depends on: [ADR-0010](0010-sops-with-age-per-cluster.md)
