<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: Apache-2.0
-->

# k8s-mgmt

<!-- markdownlint-disable MD013 -->
[![REUSE status](https://api.reuse.software/badge/github.com/wombelix/k8s-mgmt)](https://api.reuse.software/info/github.com/wombelix/k8s-mgmt)
[![Mirror](https://github.com/wombelix/k8s-mgmt/actions/workflows/mirror.yml/badge.svg)](https://github.com/wombelix/k8s-mgmt/actions/workflows/mirror.yml)
<!-- markdownlint-enable MD013 -->

## Table of Contents

* [Overview](#overview)
* [Repo layout](#repo-layout)
* [Requirements](#requirements)
* [Tasks](#tasks)
* [Runbooks](#runbooks)
* [How it works](#how-it-works)
* [Source](#source)
* [Architecture Decision Records](#architecture-decision-records)
* [Contribute](#contribute)
* [License](#license)

## Overview

This repo manages a single-node k3s cluster on a Hetzner Cloud VM,
`lab.wombacher.dev`.

* Ansible configures the node, installs k3s and bootstraps Flux.
* Flux syncs `clusters/lab-wombacher-dev/` from this repo.
* Secrets are encrypted with SOPS and age and stored in Git.

## Repo layout

* `ansible/bootstrap.yml`: runs as root after a VM rebuild. Creates the
  admin users, upgrades all packages, reboots if needed.
* `ansible/site.yml`: entry point for all other runs. Imports:
  * `ansible/node.yml`: sshd hardening, unattended-upgrades, k3s.
  * `ansible/cluster.yml`: helm on the node, Flux Operator, FluxInstance.
* `ansible/kubeconfig.yml`: copies the kubeconfig to your machine.
* `ansible/inventory.yml`: host, connection user, admin users.
* `ansible/files/`: SSH public keys, one file per account.
* `clusters/lab-wombacher-dev/`: synced by Flux. FluxInstance and the
  Flux Operator ResourceSet.
* `collections/requirements.yml`: Ansible collections. Installed into
  `collections/`, which Git ignores.
* `Taskfile.yml`: all commands.
* `.sops.yaml`: SOPS recipients.
* `.params2env.yaml`: SSM parameters for the pipeline.

## Requirements

* `ansible`, `task` and the `aws` CLI.
* An AWS login that can read `/projects/k8s-mgmt/` in SSM. Only
  `task secret` needs it.
* Your SSH key for the `wombelix` account.
* Optional: `kubectl`, `sops`, `age`.

Run `task deps` once per clone.

## Tasks

* `task deps`: installs the Ansible collections into `collections/`.
* `task bootstrap`: connects as root, creates `wombelix` and `k8s-mgmt`,
  upgrades all packages, reboots if needed. Only after a VM rebuild.
* `task site`: configures the node and bootstraps Flux. Safe to rerun.
* `task secret TARGET=<ssh destination>`: pushes the `sops-age` Secret
  from SSM. After a rebuild and after a key rotation.
* `task kubeconfig`: writes `~/.kube/lab.wombacher.dev.yaml` for local
  tools.

Arguments after `--` go to `ansible-playbook`, for example
`task site -- --check --diff`.

## Runbooks

### New or rebuilt VM

1. Rebuild the VM with a Debian 13 image, in the Hetzner console or with
   `hcloud server rebuild`.
2. `task deps`
3. `task bootstrap`
4. `task site`
5. `task secret TARGET=lab.wombacher.dev`
6. `task kubeconfig`
7. Check Flux. All objects should show `READY True`:

   ```sh
   kubectl -n flux-system get fluxinstance,kustomization,helmrelease
   ```

### Change the node

1. Edit files in `ansible/`.
2. Preview with `task site -- --check --diff`.
3. Apply with `task site`.

A second run reports `changed=0`.

### Change the cluster

Commit and push to `main`. Flux syncs `clusters/lab-wombacher-dev/`
every minute.

### Rotate the age key

1. Put your recovery key in `~/.config/sops/age/keys.txt`, so `sops` can
   decrypt the existing files.
2. Create a new cluster key and store it in SSM:

   ```sh
   umask 077 && age-keygen -o /dev/shm/k8s-mgmt.agekey
   grep '^AGE-SECRET-KEY-' /dev/shm/k8s-mgmt.agekey \
     | params2env create --path /projects/k8s-mgmt/age-private-key \
       --type SecureString --kms alias/backend-encryption \
       --region eu-central-1 --replica eu-west-1 --overwrite
   params2env create --path /projects/k8s-mgmt/age-public-key \
     --value "$(age-keygen -y /dev/shm/k8s-mgmt.agekey)" \
     --region eu-central-1 --replica eu-west-1 --overwrite
   ```

3. Replace the cluster public key in `.sops.yaml`.
4. Run `sops updatekeys <file>` for every `*.sops.yaml` file. Commit and
   push.
5. `task secret TARGET=lab.wombacher.dev`
6. `shred -u /dev/shm/k8s-mgmt.agekey`

Flux cannot decrypt between steps 4 and 5.

### Local kubeconfig

```sh
task kubeconfig
export KUBECONFIG=~/.kube/lab.wombacher.dev.yaml
```

Cluster, context and user are all `lab.wombacher.dev`. Never copy this
file to CI. Its client certificate cannot be revoked without a k3s
certificate rotation.

## How it works

### Accounts and SSH

* `wombelix`: your account for manual runs.
* `k8s-mgmt`: for the pipeline. Private key in SSM
  `/projects/k8s-mgmt/ssh-private-key-openssh`.
* Both have passwordless sudo. Public keys are in `ansible/files/`.
* `task site` turns off root login and password login. After that,
  `task bootstrap` can no longer connect as root. To rerun its tasks on
  a running node: `task bootstrap -- -e ansible_user=wombelix`.
* The pipeline connects with `-e ansible_user=k8s-mgmt`.

### Updates

* Debian: unattended-upgrades runs between 06:00 and 07:00 and reboots
  at 07:30 if needed.
* k3s: installed once from the stable channel. Later upgrades come from
  system-upgrade-controller, which is not set up yet.
* Flux: the FluxInstance uses `2.x`, so the operator upgrades Flux.
* Flux Operator: installed once with helm, then upgraded by the
  ResourceSet in `clusters/lab-wombacher-dev/flux-operator.yaml`.

### Kubernetes API

* Port 6443. The certificate covers `lab.wombacher.dev` and the public
  IPv4 and IPv6 addresses.
* `ansible/cluster.yml` runs helm and `k3s kubectl` on the node, so no
  kubeconfig leaves the node during a run.

### Secrets

* `.sops.yaml` encrypts `data` and `stringData` in every `*.sops.yaml`
  file for two age recipients:
  * the cluster key. Private key in SSM
    `/projects/k8s-mgmt/age-private-key`.
  * your personal recovery key, kept outside AWS.
* Encrypt a file with `sops encrypt --in-place my-secret.sops.yaml`.
* Flux decrypts with the `sops-age` Secret in `flux-system`.
* `task secret` pipes the private key from `aws ssm get-parameter` over
  SSH into `k3s kubectl`. The key never passes through Ansible, an
  environment variable or a file.

## Source

The primary location is:
[github.com/wombelix/k8s-mgmt](https://github.com/wombelix/k8s-mgmt)

Mirrors are available on
[Codeberg](https://codeberg.org/wombelix/k8s-mgmt) and
[Gitlab](https://gitlab.com/wombelix/k8s-mgmt).

## Architecture Decision Records

**What** was decided and **why** can be found in [docs/adr/](docs/adr/).

## Contribute

Pick the platform you prefer and are most comfortable with.

Provide feedback, open an issue or create a pull / merge request.

## License

Unless otherwise stated: `Apache 2.0`

All files contain license information either as a
`header comment` or a `corresponding .license` file.

[REUSE](https://reuse.software) from the [FSFE](https://fsfe.org/)
is implemented to verify license and copyright compliance.
