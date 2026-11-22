<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0026: App ingress with ListenerSets

## Status

Accepted

Date: 2026-11-22

## Context

[ADR-0023](0023-sidecar-less-mtls-with-istio-ambient.md) puts one shared
Istio Gateway in front of all apps. Each app has its own repo. If
every app adds a listener and a certificate to the shared Gateway, every
app needs changes in this repo.

The Gateway API has ListenerSets. A ListenerSet in an app namespace adds
listeners to a shared Gateway, and Istio supports them. The certificate
then also stays in the app namespace.

HTTP-01 challenges need port 80. cert-manager creates the challenge
route in the namespace of the certificate. No app may serve plain HTTP.

## Decision

* Apps add their HTTPS listener with a ListenerSet in their own
  namespace. The shared Gateway accepts ListenerSets from all
  namespaces.
* The Certificate, its Secret and the HTTPRoute stay in the app
  namespace. The HTTPRoute attaches to the app's ListenerSet.
* The port 80 listener on the shared Gateway accepts routes from all
  namespaces, so cert-manager's challenge routes can attach.
* A built-in ValidatingAdmissionPolicy enforces the rest:
  * Outside the Gateway's namespace, only cert-manager's challenge
    routes may attach to the shared Gateway.
  * ListenerSets may only add HTTPS listeners.
* Every other request on port 80 gets a redirect to HTTPS, as decided in
  ADR-0023.

## Consequences

* A new app needs no change in this repo.
* The cluster serves no plain HTTP, except the short-lived challenge
  responses.
* Any namespace can claim any host name. When two ListenerSets claim
  the same one, the Gateway API gives the older one priority.
* An app must use the HTTPS listener of its ListenerSet. A route on the
  shared Gateway is refused.
* ListenerSets add no pod. The shared gateway serves all listeners.
* The settings are in `infrastructure/istio-ingress/`.

## Alternatives Considered

* A listener per app on the shared Gateway: a change in this repo for
  every app.
* An HTTP listener in each app's ListenerSet for challenges: lets an
  app serve plain HTTP by mistake.
* Port 80 limited to the Gateway's namespace with Certificates there:
  certificates and listeners in this repo for every app.
* DNS-01: no port 80, but DNS provider credentials in the cluster, see
  [ADR-0017](0017-cert-manager-with-lets-encrypt.md).
* Kyverno for the validation: works only while Kyverno runs. The
  built-in policy needs no webhook.
* Only namespaces with a label may add ListenerSets: one more step per
  app for one operator.

## Related

* Amends: [ADR-0023](0023-sidecar-less-mtls-with-istio-ambient.md)
* Depends on: [ADR-0017](0017-cert-manager-with-lets-encrypt.md)
