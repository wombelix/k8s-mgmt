<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0028: Shared infrastructure in two ordered groups

## Status

Accepted

Date: 2026-11-23

## Context

[ADR-0012](0012-one-inventory-host-per-cluster.md) lets the repo manage
several clusters that share the infrastructure manifests. Each cluster
folder listed one Flux Kustomization per infrastructure component, each
with its own dependencies. A new cluster copied the whole list, and
every new component had to be added to every copy.

Some components need the custom resource definitions and webhooks of
other components before they can be applied.

The Flux reference example for several clusters groups the
infrastructure into controllers and configs. Each cluster references
the two groups.

## Decision

Each cluster applies the shared infrastructure as two ordered groups,
following the Flux reference example.

- The controllers group holds the components that install controllers
  and their custom resource definitions.
- The configs group holds the components that need them. Flux applies it
  only after the controllers group is ready.
- A new infrastructure component joins one of the two groups and reaches
  every cluster without a change to the cluster folders.
- A cluster that needs a different setting patches the shared manifests
  in its own folder.

## Consequences

- A new cluster folder references only the two groups.
- Components in one group do not wait for each other. Order inside a
  group comes from the components, for example Helm release
  dependencies.
- A config waits for all controllers, not only for the one it needs. A
  slow or failing controller delays every config.
- Flux reports the status per group. Finding the failing component takes
  one more look.
- Renaming or regrouping Flux Kustomizations on a running cluster deletes
  their resources, unless the old Kustomizations orphan them first.

## Alternatives Considered

- One Flux Kustomization per component in each cluster folder gives
  fine-grained order and status. Each cluster copies the full list.
- A shared folder with the per-component Kustomizations for all clusters
  avoids the copies and keeps the fine-grained order. It deviates from
  the Flux reference example and gives per-cluster changes no clear
  place.

## Related

- Extends: [ADR-0012](0012-one-inventory-host-per-cluster.md)
