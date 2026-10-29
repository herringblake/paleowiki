# Paleotek kluster

A self-managed Kubernetes cluster on one physical machine, **babyblue**, built with kubeadm. It was built to learn what running Kubernetes yourself involves, so it is run the way a small production cluster would be: pinned versions, GitOps, encrypted backups with rehearsed restores, monitoring that mails, and a written patching routine.

Everything is built from small, re-runnable shell scripts and manifests in the `learn_kubernetes` repository (on GitHub, `herringblake/learn_kubernetes`). These pages describe how the cluster is put together and how to operate it. They are the wiki version of that repository's README, split by topic. Where a page quotes a command, it is run from a checkout of that repository.

## At a glance

*As of 2026-10-28.*

| | |
|---|---|
| Machine | `babyblue`, 10.0.0.114 on port `enp1s0`, Ubuntu Server 26.04.1 LTS, SSH only (no monitor) |
| Kubernetes | 1.36.5 (kubeadm), one node that is both control plane and worker. End of life 2027-06-28 |
| Runtime / network | containerd 2.3.6, Cilium 1.20.2 (replaces kube-proxy; Gateway API, load-balancer addresses, Hubble) |
| Storage | local-path-provisioner: `local-nvme` (default, root disk) and `local-ssd` (second disk, `/mnt/bupgoo`) |
| Deployment | Flux 2.9 applies everything under `flux/` in the repository; deploying means committing |
| Public site | https://www.paleotek.net (nginx behind the public Gateway, Let's Encrypt certificate) |
| Internal services | `*.intranet.home.arpa` on 10.0.0.202: Grafana, Headlamp, this wiki (HTTPS from our own CA) |
| Alerts | Prometheus and Alertmanager mail `alert@paleotek.net`; a second watcher on **theopolis** covers the machine being off |
| Backups | nightly 02:30 UTC bundle on the second disk, encrypted copy pulled to theopolis |

## Where to start

**Understand it**
- [[Architecture|paleotek kluster/architecture]]: the pieces, how traffic gets in, and why they were chosen.
- [[Network and hosts|paleotek kluster/network and hosts]]: addresses, the other machines, the router.
- [[Limits and lessons|paleotek kluster/limits and lessons]]: what this setup does not protect against, and what went wrong while building it.

**Build and rebuild it**
- [[Building the cluster|paleotek kluster/building the cluster]]: the numbered scripts, in order.
- [[Rebuild from scratch|paleotek kluster/rebuild from scratch]]: start over on the same machine.
- [[Replacing the host|paleotek kluster/replacing the host]]: the machine is gone; restore onto a new one.

**Run it day to day**
- [[Maintenance calendar|paleotek kluster/maintenance calendar]]: what to do and when.
- [[Patching|paleotek kluster/patching]] and [[Upgrading Kubernetes and Cilium|paleotek kluster/upgrading kubernetes and cilium]].
- [[Monitoring and alerts|paleotek kluster/monitoring and alerts]]: what mails you, and why.
- [[Backups|paleotek kluster/backups]] and [[Restoring and drills|paleotek kluster/restoring and drills]].

**Services**
- [[Public website and HTTPS|paleotek kluster/public website and https]] and [[Certificate renewal|paleotek kluster/certificate renewal]].
- [[Intranet|paleotek kluster/intranet]], [[DNS and the Pi-hole|paleotek kluster/dns and pi-hole]] and [[Internal CA|paleotek kluster/internal ca]].
- [[Otterwiki (this wiki)|paleotek kluster/otterwiki]] and [[Headlamp|paleotek kluster/headlamp]].

**How changes are made**
- [[GitOps with Flux|paleotek kluster/gitops with flux]], [[Secrets in git|paleotek kluster/secrets in git]] and [[Flux alerts|paleotek kluster/flux alerts]].
- [[Repository and conventions|paleotek kluster/repository and conventions]] and [[Workstation access|paleotek kluster/workstation access]].

## About this wiki

The wiki is read-only in the browser. Its pages are Markdown files in the `herringblake/paleowiki` GitHub repository, and the cluster pulls that repository every 5 minutes. To change a page, edit it in that repository and push; see [[Otterwiki|paleotek kluster/otterwiki]]. Images and other attachments for a page `foo.md` go in a folder `foo/` next to it and are referenced as `./name.png`.
