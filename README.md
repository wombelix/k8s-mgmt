<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: Apache-2.0
-->

# k8s-mgmt

<!-- markdownlint-disable MD013 -->
[![REUSE status](https://api.reuse.software/badge/github.com/wombelix/k8s-mgmt)](https://api.reuse.software/info/github.com/wombelix/k8s-mgmt)
[![Mirror](https://github.com/wombelix/k8s-mgmt/actions/workflows/mirror.yml/badge.svg)](https://github.com/wombelix/k8s-mgmt/actions/workflows/mirror.yml)
[![Configure nodes with Ansible](https://github.com/wombelix/k8s-mgmt/actions/workflows/ansible.yml/badge.svg)](https://github.com/wombelix/k8s-mgmt/actions/workflows/ansible.yml)
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
  A GitHub Actions pipeline runs it on push to `main`.
* Flux syncs the cluster's folder in `clusters/` from this repo.
  The cluster folder also applies `infrastructure/`, for example
  system-upgrade-controller for k3s upgrades.
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
  `known_hosts` holds the host keys the pipeline trusts.
* `clusters/<cluster folder>/`: synced by Flux. FluxInstance, the
  Flux Operator ResourceSet and the Kustomization for
  `infrastructure/system-upgrade/`.
* `infrastructure/system-upgrade/`: system-upgrade-controller and the
  k3s upgrade Plan.
* `.github/workflows/ansible.yml`: runs `task site` on push to `main`.
* `collections/requirements.yml`: Ansible collections. Installed into
  `collections/`, which Git ignores.
* `Taskfile.yml`: all commands.
* `.sops.yaml`: SOPS recipients, one rule per cluster folder.
* `.params2env.yaml`: the repo's SSM parameters, for manual
  troubleshooting with `params2env read`. Prints secrets to stdout.

## Requirements

* `ansible`, `task` and the `aws` CLI.
* An AWS login for `/projects/k8s-mgmt/` in SSM. `task secret` reads
  from it, [Create an age key](#create-an-age-key) writes to it.
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
3. Add the host to `all.hosts` in `ansible/inventory.yml`. Don't push
   this yet, the pipeline would fail on the new host.
4. Create an age key for the cluster, see
   [Create an age key](#create-an-age-key).
   Add both of its parameters to `.params2env.yaml`.
5. Add a rule to `.sops.yaml` for `^clusters/<cluster folder>/` with the
   new public key and the recovery key. Copy the existing rule.
6. Copy an existing cluster folder to `clusters/<cluster folder>/`.
   Change `sync.path` in `flux-instance.yaml` to the new folder.
7. Commit and push everything except `ansible/inventory.yml`.
8. Follow [New or rebuilt VM](#new-or-rebuilt-vm). Its last step pushes
   the inventory.

### New or rebuilt VM

1. Rebuild the VM with a Debian 13 image, in the Hetzner console or with
   `hcloud server rebuild`. Skip this for a new VM. The rebuild keeps
   the root SSH key.
2. Remove the old host key with `ssh-keygen -R lab.wombacher.dev`. On
   the first connect, compare the fingerprint with
   `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` in the Hetzner
   console.
3. `task deps`
4. `task bootstrap -- --limit lab.wombacher.dev`
5. `task site -- --limit lab.wombacher.dev`
6. `task secret CLUSTER=lab.wombacher.dev`
7. `task kubeconfig -- --limit lab.wombacher.dev`
8. Trigger Flux. Its first run failed without the `sops-age` Secret and
   would retry only after 10 minutes. Then check that all objects show
   `READY True`:

   ```sh
   export KUBECONFIG=~/.kube/lab.wombacher.dev.yaml
   kubectl -n flux-system annotate --overwrite kustomization flux-system \
     reconcile.fluxcd.io/requestedAt="$(date +%s)"
   kubectl -n flux-system get fluxinstance,kustomization,helmrelease
   ```

9. Put the host key in `ansible/files/known_hosts`, see the comment in
   the file. Commit and push, for a new host together with
   `ansible/inventory.yml`. The pipeline run should report `changed=0`.

Until step 9, every pipeline run fails on this host.

### Change a node

1. Edit files in `ansible/`.
2. Preview with `task site -- --check --diff`.
3. Commit and push to `main`. The pipeline runs `task site`.

A local `task site` works too. A second run reports `changed=0`.

Changes to `cluster-cidr` or `service-cidr` only work on a new cluster.
Don't push them before a rebuild, run `task site` from the local change
during [New or rebuilt VM](#new-or-rebuilt-vm).

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

### Pipeline

* `.github/workflows/ansible.yml` runs on push to `main` when Ansible
  files, the Taskfile or the workflow change, and on manual dispatch.
* It assumes the repo's GitHub OIDC role and pipes the `k8s-mgmt` SSH
  key from SSM into `ssh-agent`.
* It trusts only the host keys in `ansible/files/known_hosts`. A changed
  host key fails the run.
* `task bootstrap` and `task secret` stay manual.

### Updates

Maintenance runs every night between 03:00 and 07:00 `Europe/Berlin`.
The node itself runs in UTC.

* k3s: installed once from the stable channel. system-upgrade-controller
  upgrades it to the latest stable release. Upgrades start between
  03:00 and 04:00.
* Debian: unattended-upgrades starts between 04:00 and 04:30 and
  reboots right after, if needed.
* Flux: the FluxInstance uses `2.x`, so the operator upgrades Flux.
* Flux Operator: installed once with helm, then upgraded by the
  ResourceSet `flux-operator.yaml` in the cluster folder.

### Kubernetes API

* Port 6443. The certificate covers the host name and the public IPv4
  and IPv6 addresses.
* `ansible/cluster.yml` runs helm and `k3s kubectl` on the node, so no
  kubeconfig leaves the node during a run.

### Networking

* Dual-stack, IPv4 is the primary family.
* Pods use `10.42.0.0/16` and `fd61:4655:2980::/56`, services use
  `10.43.0.0/16` and `fd61:4655:2980:100::/112`. Outgoing IPv6 from
  pods is NATed to the node address.
* The bundled Traefik is the default IngressClass `traefik`. It listens
  on ports 80 and 443 of both node addresses.

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
