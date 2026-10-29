# Secrets in git (SOPS and age)

A Secret in git is safe only when its values are encrypted. **SOPS** encrypts just the values under `data` / `stringData` and leaves keys and structure readable, so a diff still says *which* entry changed. The rules are in `.sops.yaml`: files named `flux/**/*.sops.yaml`, encrypted to an **age** public key (not a secret itself).

Flux's kustomize-controller decrypts them when it applies, using the age **private** key, which sits in the Secret `flux-system/sops-age`.

## Two different age keys

| Key | Purpose | Where the private half lives |
|---|---|---|
| **SOPS key** (the one in `.sops.yaml`) | secrets in git | the password manager (the only copy outside the cluster) **and** Secret `flux-system/sops-age` |
| **Backup key** (`AGE_RECIPIENT`) | encrypts backup bundles | the password manager **only**. It must stay off the cluster |

Anyone with cluster-admin can read `sops-age`, but they could read the live Secrets anyway. The private key is never in a file in the repository and never in `env.sh`; `test-no-personal.sh` fails a commit that contains `AGE-SECRET-KEY-`. `test-flux.sh` fails if any `kind: Secret` under `flux/` is not SOPS-encrypted, so a plain-text Secret cannot merge.

## Using it

**Setup once (workstation):** `scripts/flux/install-sops.sh` (pinned, checksum-verified, into `~/.local/bin`). To decrypt, put the private key where `sops` finds it: `export SOPS_AGE_KEY_FILE=<a file outside the repository>` while you work (delete it afterwards), or `SOPS_AGE_KEY="$(...)"` from the password manager. Encrypting needs only the public key.

**Give Flux the key:** `scripts/flux/04-sops-key.sh` (paste the private key; it is checked against `.sops.yaml` before it is stored). Run it **before** `03`, because the `apps` Kustomization asks for the Secret. Safe to re-run.

### Add a new secret

1. Write a plain Secret in a file named `flux/apps/<app>/<name>.sops.yaml` that you do **not** commit yet. Use `stringData`.
2. `sops encrypt -i flux/apps/<app>/<name>.sops.yaml`. The values become `ENC[...]`; `git diff` must show no plain text.
3. List the file under `resources:` in that app's `kustomization.yaml`, run `scripts/flux/test-flux.sh`, commit on a branch, open a pull request.
4. The Secret must not also be created by a script: Flux and `kubectl apply` would fight over it. Retire the script's step in the same pull request.

A private key or other multi-line value can go through a pipe without ever touching a file in the repository: generate it in a throwaway directory, wrap it as a `stringData` block with `sed 's/^/    /'`, pipe it through `sops encrypt --filename-override flux/apps/<app>/<name>.sops.yaml /dev/stdin`, and shred the temporary file. This is how the wiki's read-only deploy key was added (see [[Otterwiki|paleotek kluster/otterwiki]]).

### Change a value

`sops edit flux/apps/<app>/<name>.sops.yaml` (decrypts into your editor, re-encrypts on save), then commit. An app that reads the Secret as an environment variable needs a restart to see it (Alertmanager and Grafana read theirs at start). For Secrets adopted from the cluster, the files hold `data`, which is base64: put `printf %s 'new-value' | base64` in. **Never paste decrypted text into a commit message, a pull request or a chat.**

### Move a Secret that already lives in the cluster into git

`scripts/flux/06-adopt-secret.sh <namespace> <name> <folder under flux/>` writes `<folder>/<name>.sops.yaml` straight from `kubectl` through `sops`, so the values are never printed or stored in plain text. Then list the file in the folder's `kustomization.yaml`, retire whatever script created the Secret, and commit. (Set `SOPS_AGE_KEY_FILE` to also prove it decrypts to the live data.)

### What is still created by scripts

The CA's intermediate key and the Flux deploy key are created by scripts at run time, not in git.

## Gotchas

- **First encrypted file in a Kustomization that has no `decryption` block yet** (the pull request adds both): Flux may apply the new files before its own settings and send one `ReconciliationFailed` mail ("is SOPS encrypted, configuring decryption is required"). It clears on the next pass. To avoid it, add the `decryption` block in an earlier pull request.
- **Check what the cluster has:** `flux get kustomization apps` (Ready), and `kubectl -n <ns> get secret <name>`.
- **Failure modes:** `failed to get secret sops-age` means run `04`. `no matching creation rules` means the file name must end in `.sops.yaml` and sit under `flux/`. `MAC mismatch` / `failed to decrypt` means the file was edited by hand inside an `ENC[...]` value or encrypted for a different key.

## Rotating and losing the key

- **Rotate:** `age-keygen` a new pair, store the private key in the password manager, put the new public key in `.sops.yaml`, run `sops updatekeys` on each `*.sops.yaml`, run `04-sops-key.sh` with the new key, merge. If the old key **leaked**, that is not enough: the old ciphertext stays readable in git history, so treat the secrets themselves as exposed and change their values too. (A drill of the rotation has not been done yet.)
- **Rebuild of the host:** `41` restores `sops-age` with the rest of etcd, so Flux decrypts again. On a from-scratch rebuild run `04-sops-key.sh` with the password-manager key before `03`.
- **If both the cluster and the password-manager copy are gone**, the encrypted files in git cannot be recovered. The secrets must be re-created and re-encrypted with a new key.
