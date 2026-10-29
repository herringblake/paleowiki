# Headlamp (read-only web UI)

**https://headlamp.intranet.home.arpa/**, deployed by Flux from `flux/apps/headlamp/` (the chart version and values are in its `HelmRelease`; change them by committing). It shows workloads, nodes, events, logs, Gateways, certificates and so on.

**What you can do in it is decided by RBAC, not by the UI.** The buttons for editing exist, but the identity you sign in with can only read.

## Signing in

`scripts/headlamp-token.sh [duration]` prints a short-lived token (default 8 hours) for the ServiceAccount `headlamp/headlamp-viewer`. Nothing is stored. Treat the token like a password.

## What that identity can do

Defined in `flux/apps/headlamp/rbac.yaml`:

- Read most namespaced objects, using the built-in `view` role: **no Secrets, no exec, no port-forward, no changes**.
- Plus nodes, volumes, and the API groups of the tools installed here (Gateway API, cert-manager, Cilium, Prometheus operator, metrics), listed one by one and never `*`, because a wildcard would include Secrets.

`scripts/flux/verify-headlamp.sh` proves this with `kubectl auth can-i` and with a real token sent to the API server.

## The pod itself has no rights

The chart's default binds Headlamp's own ServiceAccount to `cluster-admin`. That is switched off (`clusterRoleBinding.create: false`), because everyone brings their own token.

## Not done yet

A separate identity for making changes, single sign-on (OIDC), and any deployment from the UI (the Helm feature is off).
