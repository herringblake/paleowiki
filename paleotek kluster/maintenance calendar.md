# Maintenance calendar

The recurring chores and dated deadlines are also in `maintenance.ics` in the repository; import it into a calendar. Lead times marked "suggested" are judgment calls, not deadlines.

| When | What |
|---|---|
| First Saturday of every month | **Patch window.** theopolis and the Pi-hole host: `sudo apt update && sudo apt full-upgrade`, reboot; Pi-hole: `pihole -up`. babyblue: the report first, then theopolis, the Pi-hole host, babyblue (see [[Patching|paleotek kluster/patching]]). Afterwards: DNS probes, `~/k8s-pull.log`, the machine-off watcher, Flux alerts. Review pinned versions |
| First Saturday of every third month | **Restore drills:** `restore-drill.sh` on a fresh pull, `k8s-restore-volume` into a scratch PVC, the Pi-hole import drill (see [[Restoring and drills|paleotek kluster/restoring and drills]]) |
| 2026-12-27, then every 60 days | **Check that both certificates renewed:** `scripts/flux/verify-certs.sh` shows about 90 days left on both, and `kubectl get certificates -A` a new `NOTAFTER`. The first renewal is due **2026-12-26**; the alerts mail you if one fails ([[Certificate renewal|paleotek kluster/certificate renewal]]) |
| 2027-01-16 (suggested) | Decide the theopolis OS migration date: it runs Linux Mint 21.3 on Ubuntu 22.04, whose standard support ends about April 2027 (verify the dates). Next OS: Ubuntu Server LTS, date to be set |
| 2027-04-24 | Start rehearsing the Kubernetes 1.37 upgrade |
| 2027-06-28 | Kubernetes 1.36 end of life |
| 2027-09-18 | Start renewing the API server, kubelet serving and admin kubeconfig certificates |
| 2027-10-21 | Those certificates expire |
| 2029-07-21 | Start renewing the intranet CA intermediate |
| 2029-10-24 | The intranet CA intermediate expires ([[Internal CA|paleotek kluster/internal ca]]) |

## theopolis

theopolis is an old laptop that never moves and carries five duties: the off-site copy of the cluster backups, the machine-off watcher, the restore-drill host, the DNS replica, and the pulled copies of the Pi-hole's backups (plus the Flux alert relay; see [[Flux alerts|paleotek kluster/flux alerts]]). It is on Wi-Fi with a DHCP reservation.

`scripts/theopolis/01-prepare-host.sh` (sudo, safe to re-run) makes it fit for that: lid, suspend and hibernate ignored by the system, clock synchronized, the VPN client, FTP and Apache off, Wi-Fi power saving off, the reserved address checked. `02-pihole-replica.sh` then starts the DNS replica.

**theopolis is a stopgap scheduled for replacement.** The scripts and these pages define its duties, so a new host takes them over. The cluster's Prometheus probes it (`OffsiteHostDown`).
