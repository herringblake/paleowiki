# Upgrading Kubernetes and Cilium

Both follow the policy on [[Patching|paleotek kluster/patching]]: stable, tested combinations only, rehearsed first. Both were rehearsed on 2026-10-27 in the replacement-host VM (see [[Replacing the host|paleotek kluster/replacing the host]]). **Neither has been applied to the live cluster**: Cilium 1.20.4 and Kubernetes 1.37 are deliberately not applied; the live cluster runs Cilium 1.20.2 and Kubernetes 1.36.5.

## Compatibility gate

Kubernetes upgrades are gated by Cilium. Its compatibility page lists the Kubernetes versions it is tested with: **1.33 to 1.36 on 2026-10-27, no 1.37** (Cilium 1.20.4 was the latest; 1.21 did not exist yet). The rehearsal ran Kubernetes 1.37.1 on Cilium 1.20.4 cleanly in everything used here (Gateway API, L2 announcements, kube-proxy replacement, Hubble), but that is **not a certification**, so choose knowingly. Check the Helm charts and Flux versions the same way.

## Cilium patch upgrade

*Rehearsed 2026-10-27: 1.20.2 to 1.20.4, a security patch. 1.20.3 had a regression, so go straight to 1.20.4.*

1. Read the release notes and take a fresh backup.
2. Change `CHART_VERSION` in `scripts/08-cilium.sh` and run it as your normal user. It is `helm upgrade --install` with `config/cilium-values.yaml`, then restarts the operator and agent if the config changed. (A Helm upgrade that only changes configuration does not restart Cilium's pods by itself.)
3. **Expect every Gateway (the public site, Grafana, Headlamp, the wiki) to be unreachable for about 30 to 60 seconds.** The service path and the API keep working, no pod restarts, and no alert fires (they wait minutes). It took 69 seconds.
4. Run `scripts/flux/verify-certs.sh`, `verify-www.sh`, `verify-headlamp.sh`.
5. **Rollback:** `helm rollback cilium <previous revision> -n kube-system` (find it with `helm history cilium -n kube-system`; 83 seconds, Gateways down about 70), and set the pin back.

Do it in the monthly window.

## Kubernetes minor upgrade

*Rehearsed 2026-10-27: 1.36.5 to 1.37.1, then rolled back.* One minor at a time. 1.36 reaches end of life on **2027-06-28**, so start rehearsing again from 2027-04-24.

### Before

- Read the Kubernetes release notes (removals and deprecations). `kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis` lists API versions still in use.
- Check the compatibility gate above.
- A fresh backup that theopolis has pulled.
- `scripts/patch/patch-report.sh --cluster` clean.
- Choose a window: **the API is unavailable for about two minutes** (the Gateways keep serving).

### Upgrade

On the node; `<new>` is the new package version, for example `1.37.1-1.1`.

1. `sudo systemctl start k8s-backup.service`
2. `sudo sed -i 's#stable:/v1.36/#stable:/v1.37/#' /etc/apt/sources.list.d/kubernetes.list && sudo apt-get update`
3. `sudo apt-mark unhold kubeadm && sudo apt-get install -y kubeadm=<new> && sudo apt-mark hold kubeadm`
4. `sudo kubeadm upgrade plan`. It lists what changes (in the rehearsal etcd went from 3.6 to 3.7 and CoreDNS moved; the customised Corefile was kept unchanged).
5. `sudo kubeadm upgrade apply v1.37.1 --yes`. **About 5 minutes**; the API server is down in two gaps of about 73 and 36 seconds, and the Gateways keep serving throughout.
6. `sudo apt-mark unhold kubelet kubectl && sudo apt-get install -y kubelet=<new> kubectl=<new> && sudo apt-mark hold kubelet kubectl kubeadm && sudo systemctl daemon-reload && sudo systemctl restart kubelet`. That took 8 seconds plus another brief API blip as the static pods restart. No drain was needed on this one node.

### After

- `kubectl get nodes` shows the new version and Ready. In the rehearsal no new CSR appeared; if one does, check it first.
- Wait a few minutes. Flux reconciles slowly right after the API returns, and the verify scripts can fail until it has. Cilium logs a burst of "Error retrieving lease lock" while the API is down, then stops.
- Run the verify scripts and the patch report.
- In the repository set `MINOR` in `scripts/05-kubernetes-packages.sh` and `kubernetesVersion` in `config/kubeadm-init.yaml` (for new clusters), and update the versions in the docs.

### Rollback

kubeadm cannot downgrade. The rehearsed way back, about ten minutes in all:

1. `sudo scripts/teardown/99-teardown-cluster.sh --yes`, then reboot.
2. Set `MINOR` back and run stages `10`, `20`, `30` (they install the old packages; `05` allows the downgrade), then `61`.
3. `41` with the **pre-upgrade bundle**. etcd returns to the old version because `41` uses the image its manifest names. It took 18 seconds with cached images.
4. Approve the kubelet-serving request, then run the checks.

The teardown leaves the volume directories alone, so the apps keep their live data and nothing needs a volume restore.
