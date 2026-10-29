# Network and hosts

![](./paleotek-network.drawio.png)

## The machines

| Host | Address | Role |
|---|---|---|
| madhatter (router, ASUS) | 10.0.0.1 | Gateway, DHCP, forwards ports 80 and 443 to the public Gateway. Its own DNS does **not** know the intranet names |
| pi.hole | 10.0.0.2 | Primary DNS for the LAN. Holds the `*.intranet.home.arpa` record. Static address |
| babyblue | 10.0.0.114 | The cluster node. Router DHCP reservation on port `enp1s0` |
| theopolis | 10.0.0.196 | Old laptop, Wi-Fi, DHCP reservation. Holds the off-site backup copies, the machine-off watcher, the restore-drill host, the DNS replica and the Flux alert relay. A stopgap that is scheduled for replacement |
| Printer | 10.0.0.10 | Not part of the cluster |

## Address plan (10.0.0.0/24, domain home.arpa)

| Range | Use |
|---|---|
| .1 to .19 | Static, wired devices: network hardware (.1 to .10), printers and devices (.10 to .19) |
| .20 to .199 | DHCP from the router (babyblue is .114 and theopolis .196 by reservation) |
| .200 to .219 | Virtual: the Kubernetes load-balancer pool (.200 public Gateway, .201 dev Gateway, .202 intranet Gateway) |
| .220 to .254 | Free |

Static configuration, where a device needs it: netmask 255.255.255.0, gateway 10.0.0.1, DNS 10.0.0.2 then 10.0.0.1.

Inside the cluster: pods 10.8.0.0/16, services 10.16.0.0/16. The API server is `https://babyblue:6443`, reached by hostname (the router's DNS resolves it; babyblue maps it to itself in `/etc/hosts`).

## Router settings

- DHCP range .20 to .199, with a reservation of 10.0.0.114 for babyblue's `enp1s0` and one for theopolis.
- Port forwards: 80 and 443 to 10.0.0.200 (the public Gateway). **Nothing is forwarded to .201 or .202.**
- The ISP gives a changing public IP. The DNS records for paleotek.net follow it through FreeDNS (a dynamic-DNS updater).
- A router reset or firmware update can lose the forwards; that is the first thing to check when the public site or a certificate renewal fails (see [[Certificate renewal|paleotek kluster/certificate renewal]]).
- DHCP hands out the Pi-hole (10.0.0.2) and its replica (10.0.0.196) as DNS servers and nothing else; see [[DNS and the Pi-hole|paleotek kluster/dns and pi-hole]].

## Why the node's address cannot change

The node's IP is baked into etcd, the API server's certificate, the kubelet and Cilium. A router reservation follows a port's MAC address, so moving the cable to the other network port, or changing the reservation, changes the address and breaks the cluster. Renumbering means a rebuild. `scripts/03-node-address.sh` fails early if the node is not on the expected port and address. This is how the cluster broke once, on 2026-10-16.

## Disks

| Disk | Mount | Use |
|---|---|---|
| Root (NVMe) | `/` | OS, `local-nvme` volumes in `/var/lib/k8s-volumes`, etcd |
| Second disk (SSD, 3.6 TB) | `/mnt/bupgoo` | `local-ssd` volumes in `/mnt/bupgoo/k8s-volumes`, backups in `/mnt/bupgoo/k8s-backup` |

Rule for anyone (or any tool) working on this machine: never write to, delete from or reformat the second disk outside the directories the scripts create.
