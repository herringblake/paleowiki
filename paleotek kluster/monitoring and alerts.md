# Monitoring and alerts

## The stack

`kube-prometheus-stack` (Prometheus, Alertmanager, Grafana, node-exporter, kube-state-metrics) in the `monitoring` namespace, plus the blackbox exporter, deployed by Flux from `flux/monitoring/` (controllers first, then the configs that need their CRDs).

- Prometheus keeps 15 days of data (capped at 20 GB) on `local-ssd`.
- **Grafana** is at https://grafana.intranet.home.arpa/ (LAN only; HTTPS from the [[Internal CA|paleotek kluster/internal ca]]; its login cookie is HTTPS-only).
- **Alertmanager** mails `alert@paleotek.net` from `k8s@paleotek.net` through the SMTP service at smtp.purelymail.com, port 587.
- The SMTP password and the Grafana admin password are SOPS-encrypted in `flux/monitoring/controllers/` (`alertmanager-smtp.sops.yaml`, `grafana-admin.sops.yaml`). To read the Grafana admin password: `kubectl -n monitoring get secret grafana-admin -o jsonpath='{.data.admin-password}' | base64 -d`. To change one, see [[Secrets in git|paleotek kluster/secrets in git]], "Change a value".
- `scripts/flux/verify-monitoring.sh` checks it end to end. `scripts/98-alert-test.sh` sends a test alert.

## What it tells you about, most urgent first

The rules are in `flux/monitoring/configs/alerts.yaml`.

1. **The site is down.** The blackbox exporter probes `https://www.paleotek.net/healthz` from inside the cluster through the whole public path (public DNS, router, Gateway, nginx, certificate). Critical after 3 minutes; mailed hourly until fixed. A second alert fires if the probe data itself disappears.
2. **A certificate is not renewing.** Not Ready for 15 minutes, or fewer than 21 days left (it renews at 30). See [[Certificate renewal|paleotek kluster/certificate renewal]].
3. **The backup failed or did not run.** After every run `k8s-backup.service` writes its result and the newest bundle's time to `/var/lib/node-exporter-textfile/k8s_backup.prom`, which node-exporter's textfile collector serves. Alerts: the latest run failed, no verified bundle for 26 hours, or the metrics vanished.
4. **Flux has stopped doing its job.** kube-state-metrics reports every Flux object's state as `gotk_resource_info` (custom resource state configured in `flux/monitoring/controllers/kube-prometheus-stack.yaml`; the Flux controllers do not export it in this version). Alerts: a Kustomization, HelmRelease or source not Ready for 15 minutes (warning; critical, mailed hourly, if still failing after an hour), an object suspended for an hour, a Flux controller without an available pod, or the Flux metrics gone. `dependsOn` freezes everything downstream of a failing Kustomization, so several alerts fire together; Alertmanager groups them into one mail. To change that metrics config, try it first on a throwaway kube-state-metrics pod (`--custom-resource-state-config-file`) with a broken and a suspended Flux object. For the second path that does not depend on Prometheus, see [[Flux alerts|paleotek kluster/flux alerts]].
5. **The off-site host is down.** The blackbox exporter's `ssh_banner` module greets theopolis (10.0.0.196) port 22 every minute; `OffsiteHostDown` fires after 10 minutes.
6. **DNS is failing or slow.** The blackbox exporter asks the Pi-hole, the replica, CoreDNS and, as a control, 1.1.1.1 every 30 seconds for `github.com` and `wiki.intranet.home.arpa`. Critical after 5 minutes: the Pi-hole misses the intranet name; the Pi-hole fails the public name while 1.1.1.1 works (so an internet outage is not blamed on it); or CoreDNS fails either lookup. Warnings: the replica misses the intranet name for 10 minutes; CoreDNS reports its upstream unhealthy (it fails over to public resolvers by itself, which hides a dead Pi-hole from public lookups); lookups averaging over 1 second; no DNS probe data.
7. **Patches are waiting.** `NodeSecurityRebootPending`, `NodeRebootPending`, `SecurityUpdatesNotInstalled`, `PatchMetricsMissing` (see [[Patching|paleotek kluster/patching]]).
8. **etcd.** The chart's etcd rules (no leader, fsync and commit latency, failed proposals, database size against quota, fragmentation), plus three that read what the API server reports, so they keep working if the relay breaks: database over 80% of the 2 GiB quota, p99 request latency over 250 ms, and the etcd metrics missing.

Everything else the stack ships (node disk and memory, pods crash-looping, kubelet, API server) also mails you; expect to tune the noisy ones.

## The machine being off

Nothing inside the cluster can notice the machine itself being off, so theopolis watches it. `scripts/backup/theopolis/watch-node.sh` runs from cron every 5 minutes and checks ping, the API server's `/livez`, and that the newest pulled backup copy is younger than 36 hours. A check must fail 3 runs in a row (15 minutes) before the first mail, which then repeats every 12 hours; a "resolved" mail follows recovery. It mails you directly over SMTP (Python's `smtplib`), with the SMTP login in `~/.k8s-watch.env` (mode 600) on that host only, so the alert does not depend on the cluster it reports about. `./watch-node.sh --test` proves the mail works.

**Each side watches the other:** the cluster's Prometheus probes theopolis, and theopolis watches the cluster, so one of the two always reaches you. **If both machines are down at once, nobody tells you.**

## etcd metrics

kubeadm serves etcd's metrics only on `127.0.0.1:2381`, and the controller manager and scheduler bind theirs to 127.0.0.1 as well, which a pod cannot reach. The controller manager and scheduler scrapes (and their alerts) are therefore switched off in `kube-prometheus-stack.yaml`. etcd's are on, through a systemd socket relay: `scripts/86-etcd-metrics-relay.sh` (sudo, safe to re-run) listens on the node's `cilium_host` address (the pod-network side, `ETCD_METRICS_IP`, 10.8.0.41 on 2026-10-28), port 2382, and forwards to etcd. Only pods and the node itself reach it, the LAN address refuses, and etcd is not touched or restarted.

After a rebuild the `cilium_host` address can change: set `ETCD_METRICS_IP` in `env.sh`, run 86, then `scripts/flux/01-cluster-vars.sh`; Flux upgrades the release with the new address.

The etcd database was about 34 MB, with over half of it free pages; a defrag (`etcdctl defrag`) would shrink the file but is not needed yet.

## Metrics for `kubectl top`

metrics-server (Helm chart 3.14.0, deployed by Flux from `flux/metrics-server`) serves `kubectl top nodes|pods`. It verifies the kubelet's serving certificate, which is why the kubelet requests one from the cluster CA. Until the request is approved (see [[Building the cluster|paleotek kluster/building the cluster]]) it stays NotReady, and its Kustomization is outside the dependency chain so nothing else waits. `scripts/flux/verify-metrics-server.sh` checks it. (A CSR approver add-on such as `kubelet-csr-approver` could automate the approvals later.)
