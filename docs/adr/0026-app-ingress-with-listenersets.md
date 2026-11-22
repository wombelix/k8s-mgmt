<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0026: App ingress with ListenerSets

## Status

Accepted

Date: 2026-11-22

## Context

[ADR-0027](0027-shared-istio-ingress-gateway-in-place-of-traefik.md)
puts one shared Istio Gateway in front of all apps. Each app has its own
repo. If every app adds a listener and a certificate to the shared
Gateway, every app needs changes in this repo.

The Gateway API has ListenerSets. A ListenerSet in an app namespace adds
listeners to a shared Gateway, and Istio supports them. The certificate
then also stays in the app namespace.

HTTP-01 challenges need port 80. cert-manager creates the challenge
route in the namespace of the certificate. No app may serve plain HTTP.

## Decision

Apps add their HTTPS listener to the shared Gateway with a ListenerSet
in their own namespace.

- The shared Gateway accepts ListenerSets from all namespaces.
- The Certificate, its Secret and the HTTPRoute stay in the app
  namespace. The HTTPRoute attaches to the app's ListenerSet.
- The port 80 listener on the shared Gateway accepts routes from all
  namespaces, so cert-manager's challenge routes can attach.
- A built-in ValidatingAdmissionPolicy enforces the limits. Outside the
  Gateway's namespace, only cert-manager's challenge routes may attach
  to the shared Gateway. ListenerSets may only add HTTPS listeners.
- Every other request on port 80 still gets the redirect to HTTPS.

## Consequences

- A new app needs no change in this repo.
- The cluster serves no plain HTTP, except the short-lived challenge
  responses.
- Any namespace can claim any host name. When two ListenerSets claim
  the same one, the Gateway API gives the older one priority.
- An app must use the HTTPS listener of its ListenerSet. A route on the
  shared Gateway is refused.
- ListenerSets add no pod. The shared gateway serves all listeners.

## Alternatives Considered

- A listener per app on the shared Gateway is the plain Gateway API
  way. It needs a change in this repo for every app.
- An HTTP listener in each app's ListenerSet for challenges keeps port 80
  per app. It lets an app serve plain HTTP by mistake.
- Port 80 limited to the Gateway's namespace, with all Certificates
  there, keeps challenges in one place. It needs certificates and
  listeners in this repo for every app.
- DNS-01 needs no port 80. It needs DNS provider credentials in the
  cluster, see [ADR-0017](0017-cert-manager-with-lets-encrypt.md).
- Kyverno for the validation would keep all policies in one engine. It
  works only while Kyverno runs. The built-in policy needs no webhook.
- Allowing ListenerSets only from labeled namespaces limits who can
  claim host names. It is one more step per app for one operator.

## Related

- Amends: [ADR-0027](0027-shared-istio-ingress-gateway-in-place-of-traefik.md)
- Depends on: [ADR-0017](0017-cert-manager-with-lets-encrypt.md)
