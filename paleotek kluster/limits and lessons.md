# Limits and lessons

## Limits and risks

- **Single node.** No high availability. A production control plane has three nodes for etcd quorum. Any reboot is a full outage; the plan makes it short, planned and verified ([[Patching|paleotek kluster/patching]]).
- **Local storage is tied to the node.** Requested volume sizes are not enforced. See the reclaim-policy note in [[Restoring and drills|paleotek kluster/restoring and drills]].
- **Mount ordering.** If `/mnt/bupgoo` is not mounted when a pod using `local-ssd` starts, data could land on the root disk. Accepted for a development cluster.
- **`~/.kube/config` is a cluster-admin credential.** Treat it like a root password.
- **Hubble UI has no authentication.** It is not exposed.
- **A router reservation follows a port's MAC address.** Moving the cable to the other port, or changing the reservation, changes the node's address, which is baked into the cluster. `scripts/03-node-address.sh` fails early if the node is not on the expected port and address.
- **Backups are only as good as the last drill.** The off-site copy is on theopolis, an old laptop; the cluster, its backups and the router are in one building. There is no copy outside the house.
- **Flux holds cluster-admin and a merge to `main` deploys.** The repository is the cluster's control surface.
- **Both machines down at once is silent.** Each watches the other.

## Not done yet

- Storage under Flux (StorageClasses recreated with `reclaimPolicy: Retain`).
- A rebuild rehearsal of everything that moved to Flux.
- A drill of the SOPS key rotation.
- A separate identity, or single sign-on, for Headlamp.
- Nothing alerts when the nightly Pi-hole-to-replica push stops.
- No alert watches the intranet CA intermediate's expiry (it is on the calendar instead).
- The controller manager's and scheduler's metrics are off (kubeadm binds them to localhost), so they have no alerts.
- Replacing theopolis.

## Gotchas that already cost time

**Certificates and the node**
- **Kubelet serving certificates are never auto-approved.** After a rebuild, approve them (check requester, names and IP first), or `kubectl logs/exec` and the backup fail with `tls: internal error`. With `serverTLSBootstrap` on from the start, a new kubelet has no self-signed fallback; when it was switched on in a running cluster the kubelet kept its old certificate until approval, which hid the problem.
- **A node's address is part of the cluster.** It ended up in etcd's flags and stored membership, the API certificate, the kubelet's node IP and three kubeconfigs. An endpoint alias protects clients but not the node itself, so the node needs one fixed address on one fixed port.
- **Staging before production for Let's Encrypt** (rate limits). See [[Public website and HTTPS|paleotek kluster/public website and https]].
- **The `intranet-ca` ClusterIssuer stays NotReady** until the Secret `cert-manager/intranet-ca` exists, and that holds up `infrastructure-configs` and, through `dependsOn`, every app.

**Networking**
- **Gateway API CRDs must exist before Cilium's operator starts.** Otherwise the Gateway controller never registers and the GatewayClass stays `Pending`, and a later Helm upgrade with unchanged values does not restart the operator. The CRDs go in stage 51, before Cilium. They are too large for client-side apply and need `--server-side`. After a `cilium-config` change the operator needs a restart (script 08 handles both).
- **Cilium Gateways do not separate routes by listener port.** See [[Intranet|paleotek kluster/intranet]].
- **Gateways answer ARP with the node's own MAC,** and Envoy can return 503 for a few seconds after a Gateway is Programmed.
- **Cilium's default pod range, 10.0.0.0/8, overlaps the LAN,** so the range is set explicitly (10.8.0.0/16).
- **A Cilium rollout makes every Gateway unreachable for 30 to 60 seconds,** while a `kubeadm upgrade apply` takes the API down for about two minutes but the Gateways keep serving.
- **Two DNS servers that disagree make lookups flaky.** See [[DNS and the Pi-hole|paleotek kluster/dns and pi-hole]].
- **kubeadm's CoreDNS copies the node's resolver once,** when the pod starts, so a DHCP change never reaches it.

**Storage**
- local-path-provisioner only accepts StorageClass `nodePath` values that are also listed in its ConfigMap (`nodePathMap`).
- A StorageClass cannot be edited in place, so `Retain` for new volumes waits for a deliberate recreate. `scripts/check-pv-retain.sh` catches any volume that slipped through.
- On a restored host a PV points at a directory that does not exist; the kubelet would create it root-owned 0755 and non-root apps crash. `41` creates the PV directories and the pods' subPaths with mode 0777, as local-path does.

**Host and Helm**
- A Helm upgrade that only changes configuration does not restart Cilium's pods; `08-cilium.sh` restarts them when the config changes. `helm --wait` stalled once because the control-plane taint kept Hubble's pods unscheduled, so the taint is removed before Cilium installs.
- Copying `config/` to the node only adds and overwrites; a file you renamed or deleted in the repository stays on the node.
- The kubelet package leaves a hidden `.kubelet-keep` in `/etc/kubernetes/manifests`; the restore preflight ignores it.
- Ubuntu's AppArmor stops containers talking to the host's D-Bus, so node-exporter cannot use its systemd collector; the backup publishes its own metrics file.
- `lsmod | grep -q` under `set -o pipefail` can fail with SIGPIPE even when the module is loaded; the scripts read `/proc/modules` or captured strings.
- Flux adopts a CLI-installed Helm release only if the HelmRelease has the same name and namespace.
- Flux reverts manual changes to anything it manages and prunes what you remove from git: change those objects by committing under `flux/`.

**Mistakes worth remembering**
- Sharing the dev Gateway's address with the wiki on another port (an "extra listener") answered a third of the dev site's requests with wiki pages. Backed out the same day; the intranet Gateway by hostname replaced it.
- Renumbering the node (the move to the Pi-hole network on 2026-10-16) broke the cluster, which led to the rebuild runbook, the fixed-address rule and the replacement-host rehearsals.
