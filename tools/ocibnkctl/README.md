# ocibnkctl — BNK on native k3s (host container runtime)

Container-runner module wrapping [ocibnkctl](https://github.com/mwiget/ocibnkctl), which deploys F5 BIG-IP Next for Kubernetes 2.3.1 in demo mode on a k3s cluster whose nodes run **as containers on the host docker/podman runtime** — no kind/k3d, no cloud account. Aimed at low-spec hosts where a bare-metal or DPU pipeline is overkill.

## How it runs under BNK Forge

The runner image carries `ocibnkctl` plus the tools it requires (`docker` CLI, `kubectl`, `helm`, `git`). The k3s node containers are launched as **sibling containers** on the host runtime through BNK Forge's capability-scoped docker-socket proxy (`DOCKER_HOST=tcp://docker-socket-proxy:2375`, set via `state.home_env`) — the runner never mounts the raw host socket.

## Lifecycle

- `init` (run once): `ocibnkctl init poc --customer <customer>` — creates the PoC repo at `/state/poc`
- `apply`: `ocibnkctl validate` then `ocibnkctl e2e --yolo --confirm-cluster poc` (cluster up → prereqs → FLO → shrink-if-tight → CNE; idempotent and resume-safe)
- `destroy`: `ocibnkctl destroy --yolo --confirm-cluster poc`

## Inputs

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `customer` | string | yes | Customer name recorded in the PoC repo |

## Prerequisites — read before deploying

1. **F5 entitlement files must be placed in the module state before `apply`** (delivered through F5's normal channels; there is no env/flag injection for these):
   - FAR tarball → `/state/poc/keys/` (image-pull credentials for `repo.f5.com`)
   - TEEM JWT → `/state/poc/keys/.jwt`
   Run the module's `init` first (creates `/state/poc`), drop the files into the module's state volume, then deploy.
2. **Host resource floor**: ~10 cores for the full stack (`e2e` auto-runs `deploy shrink` on tighter hosts; a 4-core `host_profile: small` exists — see the upstream README).
3. **Docker socket proxy** with container create/exec capabilities enabled (BNK Forge server installs ship `bnk-forge-docker-socket-proxy`).

## Status

First manifest revision — validated against the BNK Forge manifest contract; a full `apply` requires F5 entitlement files and has not been exercised end-to-end from a Forge deployment yet. Expect iteration (e.g. kubeconfig reachability between the runner and the k3s server container).

## Updating

New ocibnkctl release → rebuild/publish `ghcr.io/mwiget/ocibnkctl-tools-runner`, update `digest` and bump `version` in **both** `bnkforge.artifact.json` and `bnkforge.pack.json` (and any blueprint pinning this module).
