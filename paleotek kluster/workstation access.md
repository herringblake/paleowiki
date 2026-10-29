# Workstation access

`kubectl` works from the workstation against the API at `https://babyblue:6443`. The workstation needs `babyblue` to resolve (the router's DNS does this).

## Set up

- `sudo scripts/workstation/01-kubectl.sh` installs a pinned `kubectl` 1.36 from the Kubernetes apt repository.
- `scripts/workstation/02-kubeconfig.sh` copies the admin kubeconfig from babyblue to `~/.kube/config` (mode 0600). It never overwrites an existing file.
- `scripts/workstation/03-sync-to-node.sh` copies `scripts/` and `config/` to `~/k8s-setup` on babyblue, deleting what the repository no longer has. It is a **dry run unless `--yes`**. (Copying by hand only adds and overwrites, so a renamed or deleted config file would stay on the node, and `render_config` renders every file there; a stale one that uses a removed setting makes the whole script fail.)

**`~/.kube/config` is a cluster-admin credential.** Treat it like a root password; never print, commit or copy it.

## Reaching the machine

- babyblue: SSH only, no monitor, as `bherring` with key login. Sudo steps are run by the human, never by tooling.
- Hubble UI: `kubectl -n kube-system port-forward svc/hubble-ui 12000:80`, then http://localhost:12000 (not exposed otherwise; it has no authentication).
- Trust for the internal CA on each device, and the DNS caveats: [[Internal CA|paleotek kluster/internal ca]] and [[DNS and the Pi-hole|paleotek kluster/dns and pi-hole]].

## Settings live in `env.sh`

`env.sh` (git-ignored) holds every machine- and network-specific value; the template is `env.example.sh`. A copy is needed on the node, next to `scripts/`. The workstation copy is not backed up anywhere else.
