# Replacing the host

For when babyblue or its disk is gone. (The same disk surviving is the simpler case: [[Rebuild from scratch|paleotek kluster/rebuild from scratch]].)

The replacement must have the original's **name and address** (`NODE_NAME=babyblue`, `NODE_IP=10.0.0.114`, because the address is in etcd and every certificate) and a second disk mounted at `/mnt/bupgoo`.

## Rehearsed

This was rehearsed twice on 2026-10-27, in a virtual machine, from the encrypted off-site copy. The second run used the finished scripts with no manual fixes: `41` took 50 seconds, 31 pods came up by themselves, volumes came back with `--replace`, and the restored cluster took a backup of itself. The VM's times are not hardware times.

The rehearsal tool is `scripts/rehearsal/rehearsal-vm.sh`: QEMU/KVM, a private virtual network where the VM holds the node's address, and outbound mail blocked on the workstation so that the restored Alertmanager cannot send. Its commands are `create`, `start`, `sudo ... firewall on`, `check`, `snapshot`, `revert` and `destroy`. Snapshots `prepared` (clean, before the restore) and `restored` (a healthy restored cluster) are the bed for Kubernetes upgrade rehearsals; see [[Upgrading Kubernetes and Cilium|paleotek kluster/upgrading kubernetes and cilium]].

**Not proven by the VM:** real-LAN behaviour (ARP for the Gateway addresses, the router's DHCP and forwards), a fresh Let's Encrypt issuance, and real hardware.

## Procedure

1. Install Ubuntu Server, set the hostname and address, mount the second disk, and copy `scripts/`, `config/` and `env.sh` (with this host's `NODE_IFACE`) to `~/k8s-setup`.
2. `sudo ./10-prepare-system.sh`, `20-install-runtime.sh`, `30-install-kubernetes.sh`, `61-storage-dirs.sh`. About a minute; the Kubernetes packages come out at the pinned version.
3. **Fetch the newest bundle** (`k8s-backup-*.tar.gz.age` and its `.sha256`) from theopolis, into RAM (`/dev/shm`). Verify with `sha256sum -c`, then decrypt in RAM with the age private key from the password manager: `age -d -i key.txt -o /dev/shm/bundle.tar.gz <file>.age`. Delete the key and the plaintext afterwards.
4. `sudo ./41-restore-cluster.sh /dev/shm/bundle.tar.gz` (**51 seconds** to a Ready node). It also creates the volume directories, and the subPaths that pods ask for, with mode 0777 as the provisioner does. Without that, Prometheus, Alertmanager and Otterwiki crash on "permission denied".
5. `kubectl get csr`: approve the kubelet-serving request **only after checking it** (requester `system:node:babyblue`, names the node's name and address). If `41` says the port name differs, run `./72-lb-ipam.sh`.
6. Wait for the pods (images pull cold; about 5 minutes to 31 pods Running). Flux reconciles itself from git (it needs GitHub). cert-manager, both Gateways and their certificates come back from the restored Secrets with no new issuance; check with `scripts/flux/verify-certs.sh`.
7. **Volumes** (Otterwiki, Grafana): see [[Restoring and drills|paleotek kluster/restoring and drills]]. The site: `scripts/www-deploy.sh dev`, then `prod` (its content is `WWW_CONTENT` on the workstation, not in the bundle).
8. **Host-side pieces are not in the bundle:**
   - `sudo scripts/50-install-helm.sh` (the Helm release history came back with the cluster; Helm itself did not).
   - `sudo scripts/backup/install-backup-timer.sh` (needs `age` and `rsync`), then `sudo systemctl start k8s-backup.service`.
   - `k8s-restore-volume` is not installed on the new host: run `scripts/backup/restore-volume.sh` from the repository copy. The etcd metrics relay (`86`) is also host-side; run it again and then `scripts/flux/01-cluster-vars.sh` (the relay's address, `ETCD_METRICS_IP`, can change).
   - **theopolis pins the old host's SSH key and refuses the new one.** Re-pin it: `ssh-keyscan`, compare with the new `/etc/ssh/ssh_host_ed25519_key.pub`; the header of `pull-k8s-backup.sh` has the steps.
   - The workstation's kubeconfig keeps working (the CA is restored). The Pi-hole, the replica and the router are untouched.
9. **Check:** `scripts/flux/verify-certs.sh`, `verify-www.sh`, `verify-headlamp.sh`, `scripts/patch/patch-report.sh --cluster`, `scripts/dns-failover-check.sh --cluster`.

Prometheus's metrics history is not in the bundle and starts empty.
