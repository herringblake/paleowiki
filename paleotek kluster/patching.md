# Patching

**Policy: stay on stable releases and combinations that are known to work together. Move a component up only when the versions it depends on are documented as tested and the upgrade has been rehearsed. Patches, security patches included, are weighed, not rushed.** The rehearsals described here are how a version becomes "tested". Being newer is not a reason by itself.

## Four layers, each changed differently

| Layer | How it changes |
|---|---|
| **OS packages** on babyblue, theopolis and the Pi-hole host | On babyblue security fixes install by themselves (`unattended-upgrades`, release and `-security` pockets only). Everything else in the monthly window |
| **Held node packages**: `kubeadm`, `kubelet`, `kubectl`, `containerd.io`, `helm` (`apt-mark hold`) | Nothing moves them until you do |
| **Kubernetes itself** | 1.36.x patches, then 1.37 before 1.36's end of life (2027-06-28). See [[Upgrading Kubernetes and Cilium|paleotek kluster/upgrading kubernetes and cilium]] |
| **In-cluster components** (Cilium, Flux, cert-manager, charts, images) | Everything is pinned; a pin changes only when you bump it |

`unattended-upgrades` **never reboots** (`Automatic-Reboot` is off on purpose: on one node a reboot is a full outage). So a kernel or library fix can be installed but not yet running. Two tools show where things stand.

### The patch report (read-only)

- A host: `ssh <host> 'bash -s' < scripts/patch/patch-report.sh`.
- The cluster: `scripts/patch/patch-report.sh --cluster`.

It lists a host's pending reboot (how long, and whether a security update is behind it), waiting security and other updates, held packages with newer versions, the automatic-update state and, with sudo, services running old libraries. The cluster part shows nodes, pending kubelet CSRs, Flux objects and pods that are not Ready, firing alerts and the age of the newest verified backup. It ends with ACTIONS or "Nothing to do".

### The metrics and alerts (babyblue only)

An hourly timer publishes the same facts as metrics (`sudo scripts/patch/install-patch-metrics.sh` once, then `scripts/flux/verify-monitoring.sh`).

| Alert | Fires when |
|---|---|
| `NodeSecurityRebootPending` | a package that asks for the reboot came from a `-security` pocket and has waited over **1 day** (in practice, a kernel or core-library security update) |
| `NodeRebootPending` | any other pending reboot, over **7 days** |
| `SecurityUpdatesNotInstalled` | security updates have waited 2 days (`unattended-upgrades` is stuck) |
| `PatchMetricsMissing` | the metrics stopped |

theopolis and the Pi-hole host have no metrics: the monthly window and the report are their only check.

## The monthly window

First Saturday of the month; take an hour. **Order: theopolis, the Pi-hole host, babyblue.** theopolis goes first because it holds the replica, the off-site copy and the watcher. The Pi-hole host goes second because the replica answers DNS while it reboots. babyblue goes last because its outage is the largest and the others are then known good.

1. **Before:** run the report for each host and `scripts/patch/patch-report.sh --cluster`. Nothing may be firing, Flux must be Ready and the newest verified backup must be under a day old. If not, run `sudo systemctl start k8s-backup` on the node (then wait for the off-site pull) or fix the problem first.
2. **theopolis:** `sudo apt update && sudo apt full-upgrade`; reboot if the report said so. Afterwards: `docker ps` shows `pihole-replica`, `dig @10.0.0.196 wiki.intranet.home.arpa` answers, and the next pull and watcher run appear in their logs.
3. **Pi-hole host:** `sudo apt update && sudo apt full-upgrade`, then `pihole -up` if the release notes agree. **If the Pi-hole version changed, the replica must follow** (see [[DNS and the Pi-hole|paleotek kluster/dns and pi-hole]]). While it reboots run `scripts/dns-failover-check.sh --down 10.0.0.2 --cluster`, and without `--down` when it is back.
4. **babyblue:** `sudo apt update && sudo apt full-upgrade` (held packages stay). If the report or an alert says a reboot is pending, `sudo reboot`. **The whole cluster is down for a few minutes**; Flux brings everything back by itself and the volumes keep their data. Afterwards:
   - `kubectl get nodes`;
   - `scripts/patch/patch-report.sh --cluster` (alerts such as `SiteDown` may fire for a few minutes and resolve);
   - `scripts/flux/verify-certs.sh`, `verify-www.sh`, `verify-headlamp.sh`;
   - `scripts/dns-failover-check.sh --cluster`;
   - `kubectl get csr`: approve a kubelet serving request only after checking requester, names and IP.
   - Next morning: the nightly backup succeeded and `NodeRebootPending` is gone.

**An urgent reboot** (`NodeSecurityRebootPending`) is step 4 on its own, without waiting for the window: report, a fresh backup if the last is over a day old, `apt full-upgrade` if updates are still waiting, reboot, the same checks. The other hosts follow in the next window unless the update is theirs.

## Held packages

- **`containerd.io`** (and `runc` with it): read the release notes and write down the current version (`apt-cache policy containerd.io`). Then `sudo apt-mark unhold containerd.io && sudo apt install containerd.io=<version> && sudo apt-mark hold containerd.io`, then the same post-checks. containerd restarts; running pods normally keep running.
- **`kubeadm`, `kubelet`, `kubectl`** move only with the Kubernetes upgrade. Never use `apt` alone: `kubeadm upgrade` has an order, and a kubelet must never be newer than its API server.
- **In-cluster pins** (the `CHART_VERSION` line in script 08, and versions under `flux/`): read the release notes and bump one thing at a time. Use `kubectl diff` or the script's own checks first, and one commit for a Flux app. Roll back with `git revert` (Flux) or `helm rollback`.

**Rollback is not guaranteed for packages.** Older versions are often still downloadable (`apt install <package>=<version>`) but the archive does not promise it. The real safety net is the backup (etcd snapshot and volumes) and the restore drills, which is why step 1 checks the backup first.

## Not covered

Scanning container images for known vulnerabilities, firmware and BIOS updates, the router's firmware, and checking whether a newer chart or image exists (the report prints what is running; you read the release notes). A single node cannot be patched without downtime; this plan makes the downtime short, planned and verified.
