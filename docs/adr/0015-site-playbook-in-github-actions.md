<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0015: Site playbook in GitHub Actions

## Status

Accepted

Date: 2026-10-26

## Context

`task site` ran only from a local machine. The playbooks are idempotent,
so a pipeline can run them after each change. ADR-0004 reserves the
`k8s-mgmt` account and its SSH key for the pipeline.

The SSH private key is in SSM. The repo's GitHub OIDC role can read it,
and only `refs/heads/main` can assume that role. The shared
`load-ssm-secrets` action writes values to `GITHUB_ENV`, so every later
step would see the SSH key.

## Decision

The workflow `.github/workflows/ansible.yml`:

* Runs on push to `main` when Ansible files, the Taskfile or the workflow
  change, and on manual dispatch. One run at a time.
* Assumes `GitHubOIDC-<repo>-ResourceAccess` with
  `aws-actions/configure-aws-credentials`.
* Pipes the SSH key from `aws ssm get-parameter` into `ssh-add -`. Only
  the agent socket path goes into `GITHUB_ENV`.
* Trusts only the host keys in `ansible/files/known_hosts`.
* Runs `task deps` and `task site -- -e ansible_user=k8s-mgmt`, with Task
  from `go-task/setup-task`.

## Consequences

* Each merge to `main` that changes the node config applies it.
* The SSH key exists only in the agent. No file, no environment
  variable.
* After a rebuild, the run fails until `known_hosts` has the new host
  key. An unexpected host key change also fails the run.
* A new host must exist and have the `k8s-mgmt` account before its
  inventory entry is pushed.
* `task bootstrap` and `task secret` stay manual.

## Alternatives Considered

* `load-ssm-secrets` or `params2env read`: the SSH key would be in the
  environment of every later step.
* `webfactory/ssh-agent`: an extra third-party action. It takes the SSH key
  as an input, so it must first be in an environment variable or a
  GitHub secret.
* `ssh-keyscan` in the pipeline: trusts whatever host answers.
* Keep runs local only: changes depend on someone running `task site`.

## Related

* Depends on: [ADR-0004](0004-admin-accounts-per-ssh-key.md)
* Depends on: [ADR-0006](0006-taskfile-as-command-interface.md)
