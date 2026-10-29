# Internal CA (HTTPS for the intranet)

The intranet names (`*.intranet.home.arpa`) cannot get public certificates (`.home.arpa` is reserved), so the cluster has its own certificate authority, in two tiers.

| Tier | Made by | Valid | Where the key lives |
|---|---|---|---|
| **Root CA** | OpenSSL on the workstation, `scripts/ca/01-make-root.sh` | 10 years | Encrypted and **offline**: the password manager (key and passphrase as separate entries). Never in the cluster or the repository |
| **Intermediate CA** | signed by the root, `02-make-intermediate.sh` | 3 years, **expires 2029-10-24** | The only CA key in the cluster: Secret `cert-manager/intranet-ca` (installed by `03-install-ca-issuer.sh`), and in `$CA_DIR` (default `~/intranet-ca`, outside the repository) |

If the cluster is ever compromised, discard the intermediate and sign a new one; devices still trust the root.

**Name constraints:** both CA certificates are limited to `intranet.home.arpa` (an X.509 extension that browsers and OpenSSL enforce). A device that trusts the root will therefore reject any certificate this CA signs for another name. Script `02` proves it with a throwaway certificate for `www.example.com`.

**The certificate:** cert-manager issues one wildcard `*.intranet.home.arpa` (90 days, renewed 30 days before expiry) in the `intranet` namespace; the intranet Gateway's HTTPS listener uses it. The generic certificate alerts cover it. See [[Certificate renewal|paleotek kluster/certificate renewal]].

## Setting it up

In this order, `01` to `03` on the workstation (needs kubectl):

1. `scripts/ca/01-make-root.sh`: put the key and passphrase in the password manager.
2. `scripts/ca/02-make-intermediate.sh`.
3. `scripts/ca/03-install-ca-issuer.sh`: stores the Secret and creates the `intranet-ca` ClusterIssuer. Until this has run, that issuer stays NotReady, and that holds up `infrastructure-configs` and every app (see [[Rebuild from scratch|paleotek kluster/rebuild from scratch]]).
4. Flux then creates the HTTPS listener, the certificate and the redirect.
5. Trust the root on each device (below).

## Trusting the root on a device

Each device must trust the root certificate, `root.crt` (public; also published in ConfigMap `intranet/intranet-root-ca`).

- `scripts/ca/04-trust-system.sh` (sudo) does the system store (curl and so on).
- `scripts/ca/05-trust-browsers.sh` does Chrome, Chromium and Firefox (needs `libnss3-tools`).
- Both refuse a certificate that is not a CA restricted to the domain.

For another device, fetch the public root:

```
kubectl -n intranet get configmap intranet-root-ca -o jsonpath='{.data.ca\.crt}' > root.crt
openssl x509 -in root.crt -noout -fingerprint -sha256
```

Compare the fingerprint with the one `03` printed, then run `04` and `05` with that file.

## Dates to watch

The intermediate expires after 3 years and **no alert watches it** (it is not a cert-manager Certificate). Replace it before 2029-10-24 (the calendar reminds from 2029-07-21): move the old `intermediate.*` aside, then run `02` and `03`. This needs the root key from the password manager.
