<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0004: Admin accounts per SSH key

## Status

Accepted

Date: 2026-10-24

## Context

A person and a pipeline both need SSH access with sudo. Shared accounts
make it hard to tell who did what and to revoke one key. A pipeline
cannot type a sudo password. After a VM rebuild, root login and password
login are open and must be turned off.

## Decision

Each SSH key gets its own admin account with passwordless sudo, and
login is key-only.

- The account name says who or what uses it. One account is for manual
  runs by the operator, one is for the pipeline. A new person or tool
  gets its own account.
- The public keys are in the repo, one file per account. Each account
  accepts only its key from the repo. Any other key is removed.
- The pipeline's private key is in AWS SSM Parameter Store.
- Accounts have no password, so login works only with a key.
- Each account has passwordless sudo.
- The site playbook turns off root login and password login in the SSH
  server.
- The sudo and SSH server config is validated before it is written, so a
  typo cannot lock out the admins.

## Consequences

- Revoking one key means removing one file and rerunning the site
  playbook.
- Each new person or tool adds an account and a key file.
- Logs on the node show which account ran a command.
- Every account key gives full root access through sudo.

## Alternatives Considered

- One shared automation account with several keys means fewer accounts.
  No key has a clear owner.
- Sudo with a password from SSM would limit what a stolen key can do. A
  pipeline cannot enter it, and login is key-only anyway.
- The devsec.hardening collection is a broad, maintained hardening
  baseline. It is third-party, and its sysctl changes needed overrides
  for k3s. Two SSH server settings cover what we need.

## Related

- Depends on: [ADR-0003](0003-separate-bootstrap-and-site-playbooks.md)
