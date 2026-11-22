<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0011: Age key delivery over SSH, outside Ansible

## Status

Accepted

Date: 2026-10-25

## Context

Flux needs the cluster's age private key as a Secret. It is the only
secret that is not in Git. It must get from SSM to the cluster without
being written to disk or exposed to other processes.

Through Ansible, the age key would be in an environment variable, in play
and registered variables, and in the module file that Ansible writes to
the node.

## Decision

A separate task pipes the age key from SSM over SSH into kubectl on the
node, without Ansible.

- The age key moves only through pipes and a shell variable that is not
  exported. It is never in a file, an environment variable or a process
  argument.
- The task stops if the read from SSM fails or returns nothing.
- It connects with the account for manual runs by default. The pipeline
  can pass its own account.

## Consequences

- No file, environment variable, process argument or Ansible run holds
  the age key.
- It runs after a rebuild and after a key rotation, not on every site
  run. Until it runs, Flux cannot decrypt.
- The machine that runs it needs the AWS CLI and an AWS login.

## Alternatives Considered

- Ansible tasks with the age key from an environment variable would keep
  all steps in one tool. The age key leaks as described above.
- params2env reads SSM parameters. It prints `export KEY="value"`, so
  the raw value needs text parsing.
- External Secrets Operator with static IAM keys needs no manual step. A
  long-lived AWS credential becomes the bootstrap secret.
- External Secrets Operator with OIDC federation needs no bootstrap
  secret. It needs a public OIDC issuer and a JWKS update after every
  rebuild. [ADR-0020](0020-public-service-account-issuer.md) now
  provides that issuer, so this is possible later next to SOPS.

## Related

- Depends on: [ADR-0010](0010-sops-with-age-per-cluster.md)
