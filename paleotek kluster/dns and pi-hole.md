# DNS and the Pi-hole

Name lookups are the part of this network that fails in the most confusing ways, so this page records how they are meant to work.

## Who answers what

| Client | Asks | Why |
|---|---|---|
| LAN devices | the Pi-hole (10.0.0.2), then its replica on theopolis (10.0.0.196) | both hold the `*.intranet.home.arpa` records. The router's DHCP hands out only these two |
| babyblue itself | the Pi-hole alone (from DHCP) | |
| CoreDNS (inside the cluster) | the Pi-hole, then the replica, then public resolvers (1.1.1.1, 9.9.9.9) only if both stop answering | set by `scripts/87-coredns-upstream.sh` |

Every name under `intranet.home.arpa` resolves to 10.0.0.202 through one local record on the Pi-hole (Settings, All settings, expert mode, `misc.dnsmasq_lines`: `address=/intranet.home.arpa/10.0.0.202`), plus conditional forwarding of `home.arpa` to the router so its own device names still work. A new app needs a route, never a DNS change.

## The trap: two DNS servers that disagree

If a client uses a server that lacks the intranet records (the router, or a public resolver), lookups become flaky: `systemd-resolved` sticks to whichever server answered last, and the router answers NXDOMAIN for `*.intranet.home.arpa`. The cluster can be perfectly healthy and the name still fails. To tell DNS from the cluster, bypass DNS:

```
curl --resolve wiki.intranet.home.arpa:443:10.0.0.202 https://wiki.intranet.home.arpa/
```

The repository's scripts do the same (`intranet_curl` in `lib.sh`), so they work whatever the local DNS says. Give intranet records dotted names; `systemd-resolved` does not send a bare name such as `test` to a DNS server.

## CoreDNS upstream

kubeadm's default (`forward . /etc/resolv.conf`) copies the node's resolver once, when the CoreDNS pod starts. That is how the cluster kept using the flaky router (about 161 ms per lookup against 15 ms for the Pi-hole, with timeouts that failed 16 site probes in 15 days) long after the node had moved to the Pi-hole. Script 87 replaces only the `forward` and `cache` parts of the Corefile:

- `policy sequential`: a later server is used on failure, never on NXDOMAIN, so it cannot answer "no" for an intranet name.
- `serve_stale 1h` keeps recently asked names answerable through a short outage.
- kubeadm recreates the default Corefile on a rebuild, so run script 87 again after one. The old file is saved in `output/`.

## The DNS replica

One Pi-hole is a single point of failure for the whole LAN, so theopolis runs a second one.

- **What it is:** a Pi-hole container (image pinned by digest) started by `scripts/theopolis/02-pihole-replica.sh` from `pihole-replica.compose.yml`. DNS and an HTTPS API are published on 10.0.0.196 only, because `systemd-resolved` keeps 127.0.0.53.
- **How it is fed:** every night at 03:15 UTC a timer on the Pi-hole host (`pihole-sync`, installed by `scripts/pihole/install-pihole-sync.sh`) exports the Pi-hole's Teleporter archive and imports it into the replica through the replica's API, then reads the DNS records back and fails if they differ. Settings, local records, lists, groups and clients are copied. The query history never is.
- **What stays the replica's own:** its `FTLCONF_*` settings (API password, listening mode, upstreams, 7 days of history) win over the imported ones. This was checked with throwaway containers.
- **Trust:** the push trusts the replica by its certificate's public key (`curl --pinnedpubkey`), so another machine at that address gets nothing. The replica's password is in `/opt/pihole-replica/.env` (mode 600) and in `/etc/pihole-sync.pw` on the Pi-hole host.
- **When the Pi-hole recovers**, a client that stays on the replica is harmless: both answer the same.
- **Nothing alerts when the nightly push stops.** That was a decision. A stale replica shows up only through the `DnsReplicaIntranetFailing` probe, once a record is wrong. Check the push by hand with `sudo systemctl start pihole-sync.service` on the Pi-hole host.
- **Version coupling:** the replica image digest must follow the Pi-hole host's Pi-hole version. After `pihole -up`, compare `pihole -v` with `docker exec pihole-replica pihole -v`; if they differ, move the digest, re-run `02-pihole-replica.sh`, push once and repeat the import drill.

## Failover test

Failover was tested: with the Pi-hole's service stopped, and again with its cable pulled, a LAN machine, the node and CoreDNS still resolved intranet and public names (a few ms to tens of ms through the replica over Wi-Fi). Repeat it after any change to the Pi-hole, the replica or the router's DHCP:

1. Stop the Pi-hole (or pull its cable) and wait 40 seconds, so CoreDNS's 30-second cache cannot hide a failed failover.
2. `scripts/dns-failover-check.sh --down 10.0.0.2 --cluster`
3. Start the Pi-hole and run the script again without `--down`.

## Monitoring of DNS

The blackbox exporter asks the Pi-hole, the replica, CoreDNS and, as a control, 1.1.1.1 every 30 seconds, for `github.com` and for `wiki.intranet.home.arpa` (which must come back as 10.0.0.202, so a server without the records fails). Details are in [[Monitoring and alerts|paleotek kluster/monitoring and alerts]].

## Pi-hole settings backup

The Pi-hole holds the intranet records and every device depends on it, so its settings are backed up like the cluster; see [[Backups|paleotek kluster/backups]].
