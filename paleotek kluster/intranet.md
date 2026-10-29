# Intranet

Internal services share **one address, 10.0.0.202**, and are told apart by **hostname**:

| Name | Service |
|---|---|
| `grafana.intranet.home.arpa` | Grafana |
| `headlamp.intranet.home.arpa` | [[Headlamp|paleotek kluster/headlamp]], the read-only web UI |
| `wiki.intranet.home.arpa` | [[Otterwiki|paleotek kluster/otterwiki]], this wiki |

The shared Gateway lives in the `intranet` namespace (`flux/infrastructure/configs/gateways/intranet.yaml`, deployed by Flux). It serves HTTPS on 443 with a wildcard certificate from the [[Internal CA|paleotek kluster/internal ca]]; plain http on port 80 only redirects (301) to https. The router forwards nothing to this address. A namespace may attach routes to the Gateway only if it carries the label `intranet-access=true`.

## Why names, not ports or paths

Cilium keeps routes apart per Gateway and per hostname, **not per listener port**. Every route attached to a Gateway lands in one routing table, so two apps with the same match (for example `/` with no hostname) on different ports of one Gateway answer each other's requests. That was found the hard way: the dev site once answered a third of its requests with wiki pages. Paths break apps that assume they live at `/`. So: share an address by hostname (this Gateway) or give each app its own Gateway.

## Name resolution

Every name under `intranet.home.arpa` must resolve to 10.0.0.202. The Pi-hole has one local record for the whole domain, so a new app needs a route but no DNS change. How DNS is arranged, and the trap of a DNS server that lacks the record, are on [[DNS and the Pi-hole|paleotek kluster/dns and pi-hole]].

## Add an app

1. Label its namespace: `intranet-access=true`.
2. Give its HTTPRoute `parentRefs: [{name: intranet, namespace: intranet, sectionName: https}]` and `hostnames: ["app.intranet.home.arpa"]`. In a manifest under `flux/`, write the domain as `${INTRANET_DOMAIN}`.
3. Commit it; Flux applies it. Anything on a PersistentVolumeClaim under `flux/apps` needs `kustomize.toolkit.fluxcd.io/prune: disabled`.

## Trust on devices

A device that does not trust the intranet root certificate shows a certificate warning; see [[Internal CA|paleotek kluster/internal ca]].
