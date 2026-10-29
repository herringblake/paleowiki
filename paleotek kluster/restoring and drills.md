# Restoring and drills

A backup is only as good as the last time somebody restored it. This page covers the three levels of proof, from cheapest to most complete. The backups themselves are described in [[Backups|paleotek kluster/backups]].

| Level | Proves | Impact on the cluster | How often |
|---|---|---|---|
| Restore preview | what a bundle would bring back | none | whenever you wonder |
| Restore drill (on theopolis) | the off-site copy decrypts and its cluster and volume data are sound | none | every third month, and after big changes |
| Full rehearsal in a VM | the real restore procedure works on a fresh host | none | before relying on it, and before upgrades |

## Restore preview

`sudo k8s-restore-preview [bundle|latest]` starts a temporary etcd on scratch data from the bundle's snapshot, compares it with the live cluster, shows what a restore would bring back, and changes nothing.

## Restore drill (no impact on the cluster)

`scripts/backup/theopolis/restore-drill.sh <key> <bundle.age>`, run on theopolis:

1. decrypts a pulled bundle in RAM;
2. restores its etcd snapshot and runs that etcd and a kube-apiserver (images and certificates from the bundle) in an **isolated Docker network**;
3. signs in with the bundle's own admin credentials and checks the namespaces, the CA and ACME Secrets and the object counts;
4. checks the volume data in the bundle: SQLite integrity, and `git fsck` of the wiki repository.

It proves the off-site copy is restorable. It does **not** test `41-restore-cluster.sh`, kubeadm or the kubelet; the VM rehearsal does. Usage notes are in the script's header. The Pi-hole equivalent is `pihole-restore-drill.sh`.

## Restoring the cluster

A restore (`41`) brings back the cluster state, not the contents of the volumes. See [[Replacing the host|paleotek kluster/replacing the host]] for the whole procedure.

## Restoring a volume

After `41`, the PersistentVolumeClaims exist but their data does not, and the apps have already started on fresh state. For each volume:

1. **Scale the app to 0:** `kubectl -n <ns> scale deploy/<app> --replicas=0`. This works for Otterwiki and Grafana. Alertmanager and Prometheus are operator-managed, cannot be scaled by hand and do not need it (Prometheus's history is not in the bundle, and Alertmanager's silences are not worth restoring). Flux would scale an app back up; for Otterwiki's import, suspend the `apps` Kustomization first.
2. **Restore:** `sudo k8s-restore-volume <bundle> <namespace>/<pvc> --replace`. It is a **dry run until you add `--yes`**.
   - It refuses while a pod mounts the PVC.
   - With `--replace` it **moves** the fresh files to `<volume dir>.replaced-<time>` (mode 0700). Without `--replace` it refuses a directory that holds files.
   - It then restores and checks the SQLite files and the file count.
   - `--into <ns>/<pvc>` restores into a different, existing PVC. This was used for a real test: a scratch namespace, a second Otterwiki pod on the restored volume, and the same git HEAD and pages as the live wiki.
3. **Scale the app up** and delete the `.replaced-*` directory when you are satisfied.

On a replacement host `k8s-restore-volume` is not installed yet: run `scripts/backup/restore-volume.sh` from the repository copy.

Grafana's encrypted data-source values need the same `secret_key` as before; it is in the Helm values and Secrets that the etcd backup holds.

## Volume reclaim policy

The StorageClasses still say `reclaimPolicy: Delete`, which removes the data directory with the claim. The existing volumes were patched to `Retain` on 2026-10-27, so deleting a claim leaves the data on disk (the PV turns `Released`; delete its directory by hand when you are sure). After adding anything with a volume, run `scripts/check-pv-retain.sh` and patch the new PV: `kubectl patch pv <pv> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'`. Recreating the classes with `Retain` is deferred until storage moves under Flux.
