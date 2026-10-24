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
cannot type a sudo password.

## Decision

* One account per SSH key. The account name says what it is for:
  * `wombelix`: manual runs.
  * `k8s-mgmt`: the pipeline. Private key in SSM.
* Each public key is in `ansible/files/<account>.pub`. `authorized_key`
  with `exclusive: true` removes any other key.
* Both accounts have passwordless sudo, validated with `visudo`.
* Accounts have no password (`*`), login is key-only.
* `site.yml` sets `PermitRootLogin no` and `PasswordAuthentication no`
  in an sshd drop-in, validated with `sshd -t`.
* A new person or tool gets its own account.

## Consequences

* Revoking one key means removing one file and rerunning the playbook.
* Each new consumer adds an account to `admin_users`.

## Alternatives Considered

* One shared automation account with several keys: no clear owner per
  key.
* Sudo with a password from SSM: a pipeline cannot enter it, and login
  is key-only anyway.
* `devsec.hardening` collection: third-party, and its sysctl changes
  needed overrides for k3s. Two sshd settings are enough here.

## Related

* Depends on: [ADR-0003](0003-bootstrap-and-site-playbooks.md)
