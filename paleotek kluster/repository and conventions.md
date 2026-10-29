# Repository and conventions

Everything is in one repository, `herringblake/learn_kubernetes` on GitHub. Its README is the original, longer-form tour; these wiki pages are the same material split by topic.

## Layout

| Path | Contents |
|---|---|
| `scripts/` | Numbered step scripts, stage scripts and `lib.sh`. Subfolders: `flux/` (install Flux, SOPS keys, the verify scripts, `test-flux.sh`), `backup/` (the nightly bundle, restore preview, volume restore; `backup/theopolis/` runs on theopolis), `pihole/` and `theopolis/` (Pi-hole backup and sync, the DNS replica, the Flux alert relay), `ca/` (the internal CA), `patch/` (patch report and metrics), `rehearsal/` (the throwaway replacement-host VM), `teardown/` (tear the cluster down for a rebuild), `workstation/` |
| `flux/` | Everything Flux deploys; see [[GitOps with Flux|paleotek kluster/gitops with flux]] |
| `config/` | What the scripts apply: kubeadm config, Cilium Helm values, pinned upstream manifests (Gateway API CRDs, local-path-provisioner), the StorageClasses, the load-balancer pool |
| `www/` | A sample site (the real one is `WWW_CONTENT` in `env.sh`) |
| `env.example.sh` | Template for `env.sh`, the settings file (git-ignored) |
| `.github/workflows/ci.yml` | CI |
| `maintenance.ics` | The calendar; see [[Maintenance calendar|paleotek kluster/maintenance calendar]] |
| `packages/`, `NETWORKCFG.md`, `www-local/` | Private reference material (router manual, address plan, the real site); git-ignored |

## Settings and templates

`env.sh` holds every machine- and network-specific value (node name, IP and port, address ranges, public hostname, disk paths, backup schedule). Files under `config/` use `{{NAME}}` placeholders; each script that applies one fills them from `env.sh` at run time (`render_config` in `scripts/lib.sh`) into a temporary directory and fails on an unset name. Scripts apply from that temporary copy, never from `config/` directly. After changing placeholders, run `scripts/test-render.sh`.

Under `flux/`, settings are `${NAME}` placeholders filled from the cluster ConfigMap `cluster-vars`.

## Conventions for scripts

- Each script is **idempotent**, fails early with a message saying what to do, and ends with a **verification step that proves the outcome** (not just that a command ran).
- `sudo` is outermost (`sudo ./stage.sh`), never nested in a pipeline or called from another script.
- Destructive scripts print a dry run unless given `--yes`.
- Verify config changes against the live cluster before applying: render, then `kubectl diff` (no output means no change).
- Bash: no `| grep -q` under `pipefail` (SIGPIPE); read `/proc/modules` rather than piping `lsmod`; retry anything eventually consistent (Gateway serving, certificates); trap commands expand variables at set time; use `BASH_SOURCE` when a file may be sourced.

## Checks and CI

CI runs on every pull request and every push to `main`, offline, with `env.example.sh` as the environment:
- `bash -n` on every script;
- `scripts/test-render.sh`;
- `scripts/flux/test-flux.sh` (builds the Flux manifests, checks every variable resolves, rejects a plain `kind: Secret`, requires `prune: disabled` on PVCs);
- `scripts/patch/test-patch-metrics.sh`;
- `scripts/test-no-personal.sh`: no LAN addresses, real e-mail addresses or obvious secrets in tracked files (with `env.sh` present it also looks for every address, e-mail and domain in it).

Run the same checks locally before pushing. Work happens on feature branches merged by pull request (no approval required). In committed comments and docs use placeholders such as `<NODE_IP>`, never the real addresses. (This wiki is the place for the real ones.)

## Safety rules

- Never write to, delete from or reformat the second disk outside the directories the scripts create.
- Anything on ports 80/443 of the public Gateway is on the internet. Do not add hostnames or routes there casually.
- Backup bundles, the admin kubeconfig and the intranet CA's keys are secrets. The CA root key stays offline in the password manager; the intermediate lives in a Secret and in `$CA_DIR`.
- Keep personal values (addresses, domains, emails) out of committed files in the repository.
- Upgrades follow the policy in [[Patching|paleotek kluster/patching]]: stable, tested combinations only; never bump a pin (`CHART_VERSION`, `MINOR`, image tags) without a decision.
