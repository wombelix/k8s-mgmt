<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0015: Site playbook in GitHub Actions

## Status

Accepted

Date: 2026-10-26

## Context

The site playbook ran only from a local machine. The playbooks are
idempotent, so a pipeline can run them after each change.
[ADR-0004](0004-admin-accounts-per-ssh-key.md) reserves an account and
its SSH key for the pipeline.

The SSH private key is in SSM. The repo's GitHub OIDC role can read it,
and only the main branch can assume that role. The shared action that
loads SSM parameters writes them to the job environment, so every later
step would see the SSH key.

## Decision

A GitHub Actions workflow runs the site playbook after each relevant
change on the main branch.

- It runs on push to main when Ansible files, the Taskfile or the
  workflow change, and on manual dispatch. Only one run happens at a
  time.
- It assumes the repo's OIDC role with the official AWS credentials
  action. No AWS keys are stored in GitHub.
- It pipes the SSH key from the AWS CLI into ssh-agent. Later steps get
  only the agent socket.
- It trusts only the host keys pinned in the repo.
- It runs the same tasks as a person, as the pipeline account.

## Consequences

- Each merge to main that changes the node config applies it.
- The SSH key exists only in the agent, never in a file or an
  environment variable.
- After a rebuild, the run fails until the repo has the new host key. An
  unexpected host key change also fails the run.
- A new host must exist and have the pipeline account before its
  inventory entry is pushed.
- The bootstrap playbook and the age key delivery stay manual.

## Alternatives Considered

- The shared SSM action or `params2env read` would reuse existing
  tools. The SSH key would be in the environment of every later step.
- The `webfactory/ssh-agent` action is a common choice. It is one more
  third-party action, and it takes the SSH key as an input. The SSH key
  must first be in an environment variable or a GitHub secret.
- `ssh-keyscan` in the pipeline needs no host key in the repo. It trusts
  whatever host answers.
- Keeping runs local needs no pipeline. Changes then depend on someone
  running the site playbook.

## Related

- Depends on: [ADR-0004](0004-admin-accounts-per-ssh-key.md)
- Depends on: [ADR-0006](0006-taskfile-as-command-interface.md)
