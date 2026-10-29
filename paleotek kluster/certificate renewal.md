# Certificate renewal

Two certificates renew by themselves, each **30 days before expiry**. Let's Encrypt no longer sends expiry reminders, so the alerts below are the only warning.

| Certificate | Issuer | Lifetime | Expires | Renews from |
|---|---|---|---|---|
| `public/www-paleotek-net-tls` (www.paleotek.net) | Let's Encrypt, HTTP-01 through the public Gateway | 90 days | 2027-01-25 | 2026-12-26 |
| `intranet/intranet-wildcard` (`*.intranet.home.arpa`) | our own [[Internal CA|paleotek kluster/internal ca]] | 90 days | 2027-01-25 | 2026-12-26 |

*Dates read from the cluster on 2026-10-28.* To read them again:

```
kubectl get certificates -A -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,READY:.status.conditions[0].status,NOTAFTER:.status.notAfter,RENEWAL:.status.renewalTime
```

Other certificates that expire: the API server, kubelet serving and admin kubeconfig certificates on **2027-10-21**, and the intranet CA's intermediate on **2029-10-24**. Both are in the [[Maintenance calendar|paleotek kluster/maintenance calendar]].

## Alerts

| Alert | Meaning |
|---|---|
| `CertificateNotReady` | cert-manager has reported a certificate not Ready for 15 minutes |
| `CertificateRenewalStuck` (warning) | still under 29 days from expiry 12 hours on, about 1.5 days after the first attempt |
| `CertificateExpiringSoon` (critical) | under 21 days left |

The alert rules assume the 30-day renewal window. If a `renewBefore` or a certificate lifetime changes, change the `29` in `flux/monitoring/configs/alerts.yaml`. The calendar also has a check on the day after each renewal is due, every 60 days.

## Rehearse a renewal

Do this once before the first real renewal, and again after any change to the router, the DNS records, the public Gateway or cert-manager. It runs the real path: Let's Encrypt validators on the internet → public IP → router → Gateway → solver pod. It is safe: cert-manager keeps serving the current certificate until the new one is issued, so a failed rehearsal changes nothing for visitors. **Each public run uses one of Let's Encrypt's 5 duplicate certificates per week**, so do not repeat it more than twice a week.

You need `cmctl`, cert-manager's CLI, in the same minor version as the controller (find it with `kubectl -n cert-manager get deploy cert-manager -o jsonpath='{.spec.template.spec.containers[0].image}'`). Download it from the cert-manager/cmctl releases on GitHub and check it against the release's checksum file.

1. **Note the starting point.** `kubectl -n public get certificate` (the name ends in `-tls`), and `-o jsonpath='{.status.revision} {.status.notAfter}{"\n"}'`. Note the serial being served: `echo | openssl s_client -connect 10.0.0.200:443 -servername www.paleotek.net 2>/dev/null | openssl x509 -noout -serial -enddate`.
2. **Renew:** `cmctl renew -n public www-paleotek-net-tls`.
3. **Watch:** `kubectl -n public get certificate,certificaterequest,order,challenge` every few seconds. The order is: Certificate `Issuing`, a CertificateRequest, an Order `pending`, a Challenge `pending` (20 to 60 seconds is normal: the solver pod and its temporary HTTPRoute are created, and Let's Encrypt fetches the token), the Order `valid`, the Certificate Ready again. One to three minutes in all.
4. **Prove it:** the Certificate is Ready, its `revision` went up by one, and `notAfter` is about 90 days away. The openssl command now shows a different serial and the new end date (the Gateway picks up the new Secret within seconds). `scripts/flux/verify-certs.sh` passes.
5. **The intranet certificate** has no outside dependency: `cmctl renew -n intranet intranet-wildcard`, then `verify-certs.sh`.
6. **Renewal times move** to about 30 days before the new expiry. Read the new dates with the command above. The calendar's renewal check repeats every 60 days (a 90-day certificate renewed 30 days early) and was aligned to the 2026-10-27 rehearsal. If a rehearsal or a manual renewal shifts the dates, move its start date in `maintenance.ics`.

## If a renewal fails

Start with `kubectl -n public describe certificate <name>`, then `kubectl -n public get certificaterequest,order,challenge` and `describe` the pending one. The Challenge's `Reason` says what Let's Encrypt saw. cert-manager retries with a growing delay (hours), so after a fix run `cmctl renew` again.

| Symptom | Check, in this order |
|---|---|
| Timed out, connection refused, or the wrong content (the validator cannot reach the solver) | From outside the network (a phone on mobile data), `curl -v http://www.paleotek.net/.well-known/acme-challenge/test` must reach the cluster; an Envoy or nginx 404 is fine, a hang is not. Then: (1) the router still forwards port 80 and 443 to 10.0.0.200 (a router reset or firmware update loses it); (2) the DNS record still points at the current public IP (`dig +short www.paleotek.net @1.1.1.1` against a "what is my IP" site; the dynamic-DNS updater may have stopped); (3) the ISP has not started blocking port 80. From inside the network this test can mislead, because the router may answer for the name itself |
| The solver exists but is not served | `kubectl -n public get gateway public` must be Programmed; `kubectl -n public get httproute` shows the temporary `cm-acme-http-solver` route; Envoy can return 503 for a few seconds after a change |
| `rateLimited` | Wait the time in the message and do nothing else. Every retry and every rehearsal counts against the limit |
| ClusterIssuer not Ready, or an account error | `kubectl get clusterissuer letsencrypt-prod` and the Secret `cert-manager/letsencrypt-prod-account-key` (a restore brings it back; a new account is registered if it is gone, which is fine). Read `kubectl -n cert-manager logs deploy/cert-manager`; if `logs` fails with `tls: internal error`, approve the kubelet serving CSRs first |
| The intranet certificate | Its ClusterIssuer `intranet-ca` must be Ready, which needs the Secret `cert-manager/intranet-ca` (`scripts/ca/03`) |

**Do not:** delete the certificate's Secret while it is still valid (the Gateway would lose its certificate at once); point the Gateway at the staging issuer (browsers distrust staging certificates); or retry in a loop.

The old certificate keeps working until its `notAfter`, so an alert at 28 days leaves four weeks. If it cannot be fixed before expiry, visitors see a browser warning on the public site and nothing else breaks (the intranet has its own CA; mail and GitHub do not use this certificate). Accept the warning rather than loosen the setup, and fix it properly.
