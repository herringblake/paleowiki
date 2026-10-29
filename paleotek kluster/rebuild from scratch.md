# Rebuild from scratch

Use this to start over **on the same machine and disk**, for example after changing the node's address. If the machine or its disk is gone, use [[Replacing the host|paleotek kluster/replacing the host]] instead.

What survives a rebuild: backup bundles, containerd and its cached images, the installed packages and the volume directories.

## Steps

1. **Preview the teardown.** `sudo scripts/teardown/99-teardown-cluster.sh` is a dry run that prints every action. Read it.
2. **Tear down.** `sudo scripts/teardown/99-teardown-cluster.sh --yes`, then **reboot** (this clears Cilium's kernel state).
3. **Run the stages in order**, as described in [[Building the cluster|paleotek kluster/building the cluster]]: `10` (it fails early if `enp1s0` does not hold 10.0.0.114), `20` and `30` (quick, everything is already installed), then `40`.
4. **Approve the kubelet's serving certificate right after `40`.** The rebuilt kubelet asks for one from the start and has no fallback. Until you approve it, `kubectl logs`, `exec` and `port-forward` fail, and so does the nightly backup. Check requester, names and IP of each request first.
5. **Run `50`, `51`, `61` followed by `60`, and `70`.** Run `87-coredns-upstream.sh` once CoreDNS is up, because kubeadm recreates its default Corefile.
6. **Hand the cluster to Flux.** Remove the test app (`73-gateway-test.sh --cleanup`, which frees 10.0.0.200). Then run `scripts/flux/01-cluster-vars.sh`, `02`, `04` (the SOPS key) and `03`, in that order. `04` must come before `03`, because the `apps` Kustomization decrypts with the Secret it creates. All settings must be in `env.sh` first. Flux then installs cert-manager, the issuers, both Gateways, Headlamp, Otterwiki and the nginx sites in dependency order.
7. **Install the intranet CA's key.** It is not in git. Run `scripts/ca/03-install-ca-issuer.sh` with the CA files from the password manager. Until then the `intranet-ca` issuer stays NotReady, and that holds up `infrastructure-configs` and, through `dependsOn`, every app (they retry every two minutes).
8. **Put the content in.** The volumes start empty: `www-deploy.sh dev`, then `prod`. Restore Otterwiki's and Grafana's data as described in [[Restoring and drills|paleotek kluster/restoring and drills]] (Otterwiki's pages also come back from the paleowiki repository on the next pull).
9. **Workstation.** Move the old `~/.kube/config` aside (`02-kubeconfig.sh` never overwrites), remove any stale `nauplius` line from `/etc/hosts`, and run `scripts/workstation/02-kubeconfig.sh`.
10. The nightly backup resumes by itself once the API is up. The timer lives on the host's disk, which this procedure keeps. On a replacement host it does not.

## Let's Encrypt on a from-scratch rebuild

A rebuild with no restored Secrets has no certificate and would request a new production one straight away. Production allows only 5 duplicate certificates per week. Before step 6, commit `letsencrypt-staging` as the issuer annotation on the public Gateway (`flux/infrastructure/configs/gateways/public.yaml`), check that it works (browsers warn about a staging certificate, as expected), then commit the switch back to `letsencrypt-prod`. See [[Public website and HTTPS|paleotek kluster/public website and https]].
