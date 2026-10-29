# Public website and HTTPS

**https://www.paleotek.net** is served by the **public Gateway** at 10.0.0.200. The router forwards ports 80 and 443 to that address, so anything on those two ports is reachable from the internet.

> Do not add hostnames or routes to the public Gateway without a deliberate decision. Both listeners answer only for `www.paleotek.net`; any other hostname gets a 404.

## The path of a request

internet → DNS record (follows the changing public IP through FreeDNS) → router (forwards 80 and 443) → public Gateway 10.0.0.200 (TLS ends here) → nginx in namespace `www`.

- **nginx** is `nginxinc/nginx-unprivileged` (stable line, pinned). Its settings are in `flux/apps/www/base/default.conf`, a ConfigMap whose name carries a hash, so editing it restarts the pod. The Content-Security-Policy allows only same-origin scripts: no inline `<script>`, no external fonts or CDNs.
- **Content** lives on a PersistentVolumeClaim and is copied in by `scripts/www-deploy.sh prod`. The real site files are on the workstation (`www-local/`, `WWW_CONTENT` in `env.sh`); the repository's `www/` is only a sample.
- **Dev copy:** the same site in namespace `www-dev`, served as plain http at http://10.0.0.201/ (a second Gateway, LAN only, because the router forwards nothing to that address). Promote by deploying to dev, checking, then deploying to prod.
- To put a different service behind the name later, change the backend in `flux/apps/www/prod/routes.yaml` (a commit).

## The certificate

cert-manager issues the certificate from Let's Encrypt with the **HTTP-01** challenge (one certificate per name, no DNS API needed) and renews it by itself, 30 days before expiry. The certificate is `public/www-paleotek-net-tls`.

**Staging versus production** is one line: the annotation `cert-manager.io/cluster-issuer` on the public Gateway in `flux/infrastructure/configs/gateways/public.yaml`, committed as `letsencrypt-prod`. Switching between `letsencrypt-staging` and `letsencrypt-prod` is a commit. Production allows 5 duplicate certificates per week and 50 per domain per week.

- A restore from an etcd backup brings the certificate Secret back, so nothing is re-issued.
- A from-scratch rebuild has no Secret and would ask for a new production certificate immediately: commit `letsencrypt-staging` first, check it, then commit the switch back (browsers warn about a staging certificate, which is expected).
- The ACME account keys are Secrets in the `cert-manager` namespace (`letsencrypt-*-account-key`). The contact address is `ACME_EMAIL` in `env.sh`.
- The account key and certificate Secrets live in etcd, so they are in the (root-only) backup bundles.

## What it depends on outside the cluster

- The DNS records for paleotek.net must point at the current public IP. Something on the router side updates them; if it stops, the site and renewals fail.
- The router must forward port 80 (and 443) to 10.0.0.200.
- The ISP must not block port 80.

Internal-only services cannot get certificates this way, because Let's Encrypt must reach port 80 from the internet. They use the [[Internal CA|paleotek kluster/internal ca]]. Make sure the router's own local domain (`home.arpa`) is never the same as the public one, or the router would answer for the public names inside the network.

## Watching it

The blackbox exporter probes `https://www.paleotek.net/healthz` from inside the cluster, through the whole public path. `SiteDown` is critical after 3 minutes. See [[Monitoring and alerts|paleotek kluster/monitoring and alerts]]; for renewal, [[Certificate renewal|paleotek kluster/certificate renewal]].
