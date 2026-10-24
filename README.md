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

This repo manages single-node k3s clusters on Hetzner Cloud VMs. Each
host in `ansible/inventory.yml` is one cluster.

* Ansible configures the node, installs k3s and bootstraps Flux.
* Flux syncs the cluster's folder in `clusters/` from this repo.
* Secrets are encrypted with SOPS and age and stored in Git. Each cluster
  has its own age key.

Current clusters: `lab.wombacher.dev`.

The examples below use `lab.wombacher.dev`. Replace it with your host.
A host's cluster folder is its name with dashes instead of dots, for
example `clusters/lab-wombacher-dev/`.

## Repo layout

* `ansible/bootstrap.yml`: runs as root after a VM rebuild. Creates the
  admin users, upgrades all packages, reboots if needed.
* `ansible/site.yml`: entry point for all other runs. Imports:
  * `ansible/node.yml`: sshd hardening, unattended-upgrades, k3s.
  * `ansible/cluster.yml`: helm on the node, Flux Operator, FluxInstance.
* `ansible/kubeconfig.yml`: copies the kubeconfig to your machine.
* `ansible/inventory.yml`: one line per cluster host. Shared settings
  are in `all.vars`.
* `ansible/files/`: SSH public keys, one file per account.
* `clusters/<cluster folder>/`: synced by Flux. FluxInstance and the
  Flux Operator ResourceSet.
* `collections/requirements.yml`: Ansible collections. Installed into
  `collections/`, which Git ignores.
* `Taskfile.yml`: all commands.
* `.sops.yaml`: SOPS recipients, one rule per cluster folder.
* `.params2env.yaml`: SSM parameters for the pipeline.

## Requirements

* `ansible`, `task` and the `aws` CLI.
* An AWS login that can read `/projects/k8s-mgmt/` in SSM. Only
  `task secret` needs it.
* Your SSH key for the `wombelix` account.
* Optional: `kubectl`, `sops`, `age`, `params2env`.

Run `task deps` once per clone.

## Tasks

* `task deps`: installs the Ansible collections into `collections/`.
* `task bootstrap`: connects as root, creates `wombelix` and `k8s-mgmt`,
  upgrades all packages, reboots if needed. Only after a VM rebuild.
* `task site`: configures the node and bootstraps Flux. Safe to rerun.
* `task secret CLUSTER=<host>`: pushes the cluster's `sops-age` Secret
  from SSM. After a rebuild and after a key rotation. `SSH_USER`
  defaults to `wombelix`.
* `task kubeconfig`: writes `~/.kube/<host>.yaml` for local tools.

`bootstrap`, `site` and `kubeconfig` run against all hosts. Arguments
after `--` go to `ansible-playbook`. Limit a run to one cluster with
`--limit`, for example:

```sh
task site -- --limit lab.wombacher.dev --check --diff
```

## Runbooks

### Add a cluster

1. Create a Hetzner Cloud VM with a Debian 13 image and your SSH key for
   root. Turn on Backups and delete protection. Keep rebuild protection
   off.
2. Set DNS: A and AAAA records for the host, and the reverse DNS of both
   IPs to the host name.
3. Add the host to `all.hosts` in `ansible/inventory.yml`.
4. Create an age key for the cluster, see
   [Create an age key](#create-an-age-key).
5. Add a rule to `.sops.yaml` for `^clusters/<cluster folder>/` with the
   new public key and the recovery key. Copy the existing rule.
6. Copy an existing cluster folder to `clusters/<cluster folder>/`.
   Change `sync.path` in `flux-instance.yaml` to the new folder.
7. Commit and push.
8. Follow [New or rebuilt VM](#new-or-rebuilt-vm).

### New or rebuilt VM

1. Rebuild the VM with a Debian 13 image, in the Hetzner console or with
   `hcloud server rebuild`. Skip this for a new VM.
2. `task deps`
3. `task bootstrap -- --limit lab.wombacher.dev`
4. `task site -- --limit lab.wombacher.dev`
5. `task secret CLUSTER=lab.wombacher.dev`
6. `task kubeconfig -- --limit lab.wombacher.dev`
7. Check Flux. All objects should show `READY True`:

   ```sh
   export KUBECONFIG=~/.kube/lab.wombacher.dev.yaml
   kubectl -n flux-system get fluxinstance,kustomization,helmrelease
   ```

### Change a node

1. Edit files in `ansible/`.
2. Preview with `task site -- --check --diff`.
3. Apply with `task site`.

A second run reports `changed=0`.

### Change a cluster

Edit the cluster's folder in `clusters/`, commit and push to `main`.
Flux syncs it every minute.

### Create an age key

Run this for a new cluster or a key rotation. Replace the host in
`P=...`:

```sh
P=/projects/k8s-mgmt/lab.wombacher.dev
umask 077 && age-keygen -o /dev/shm/cluster.agekey
grep '^AGE-SECRET-KEY-' /dev/shm/cluster.agekey \
  | params2env create --path "$P/age-private-key" \
    --type SecureString --kms alias/backend-encryption \
    --region eu-central-1 --replica eu-west-1 --overwrite
params2env create --path "$P/age-public-key" \
  --value "$(age-keygen -y /dev/shm/cluster.agekey)" \
  --region eu-central-1 --replica eu-west-1 --overwrite
age-keygen -y /dev/shm/cluster.agekey
shred -u /dev/shm/cluster.agekey
```

The last `age-keygen` prints the public key for `.sops.yaml`.

### Rotate a cluster age key

1. Put your recovery key in `~/.config/sops/age/keys.txt`, so `sops` can
   decrypt the existing files.
2. [Create an age key](#create-an-age-key) for the cluster.
3. Replace the cluster public key in the cluster's rule in `.sops.yaml`.
4. Run `sops updatekeys <file>` for every `*.sops.yaml` file in the
   cluster folder.
5. Commit and push.
6. `task secret CLUSTER=lab.wombacher.dev`

Flux cannot decrypt between steps 5 and 6.

### Local kubeconfig

```sh
task kubeconfig -- --limit lab.wombacher.dev
export KUBECONFIG=~/.kube/lab.wombacher.dev.yaml
```

Cluster, context and user all use the host name. Never copy this file to
CI. Its client certificate cannot be revoked without a k3s
certificate rotation.

## How it works

### Accounts and SSH

* `wombelix`: your account for manual runs.
* `k8s-mgmt`: for the pipeline. Private key in SSM
  `/projects/k8s-mgmt/ssh-private-key-openssh`, the same on all hosts.
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
  ResourceSet `flux-operator.yaml` in the cluster folder.

### Kubernetes API

* Port 6443. The certificate covers the host name and the public IPv4
  and IPv6 addresses.
* `ansible/cluster.yml` runs helm and `k3s kubectl` on the node, so no
  kubeconfig leaves the node during a run.

### Secrets

* Each cluster has its own age key. The private key is in SSM at
  `/projects/k8s-mgmt/<host>/age-private-key`.
* `.sops.yaml` has one rule per cluster folder. It encrypts `data` and
  `stringData` in `*.sops.yaml` files for two recipients:
  * the cluster key
  * your personal recovery key, kept outside AWS
* Files outside a cluster folder match no rule, so `sops` refuses to
  encrypt them.
* Encrypt a file from the repo root:

  ```sh
  sops encrypt --in-place clusters/lab-wombacher-dev/my-secret.sops.yaml
  ```

* Flux decrypts with the `sops-age` Secret in `flux-system`.
* `task secret` pipes the private key from `aws ssm get-parameter` over
  SSH into `k3s kubectl`. It never passes through Ansible, an
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
