# ocibnkctl — BNK on native k3s (host container runtime)

Container-runner module wrapping [ocibnkctl](https://github.com/mwiget/ocibnkctl) **v2.3.1-10**, which deploys F5 BIG-IP Next for Kubernetes 2.3.1 in demo mode on a k3s cluster whose nodes run **as containers on the host docker/podman runtime** — no kind/k3d, no cloud account. Aimed at low-spec hosts where a bare-metal or DPU pipeline is overkill.

## How it runs under BNK Forge

The runner image carries `ocibnkctl` plus the tools it requires (`docker` CLI, `kubectl`, `helm`, `git`). The k3s node containers are launched as **sibling containers** on the host runtime through BNK Forge's capability-scoped docker-socket proxy (`DOCKER_HOST=tcp://docker-socket-proxy:2375`, set via `state.home_env`) — the runner never mounts the raw host socket.

### Runner image provenance

The pinned image `ghcr.io/mwiget/ocibnkctl-tools-runner` (multi-arch: linux/amd64 + linux/arm64) is built and published from the upstream **ocibnkctl** repo, not from this catalog:

- **Build source:** [`runner.Dockerfile`](https://github.com/mwiget/ocibnkctl/blob/main/runner.Dockerfile) — `FROM alpine`, installs the tool set, and downloads + checksum-verifies the `ocibnkctl` binary from the matching GitHub release.
- **Who builds it:** the ocibnkctl maintainer, as a step in the release runbook ([`docs/RELEASE.md`](https://github.com/mwiget/ocibnkctl/blob/main/docs/RELEASE.md)) via `make runner-image RUNNER_VERSION=<ver> RUNNER_PLATFORM=linux/amd64,linux/arm64 PUSH=1`. It is a manual publish step — ocibnkctl CI (goreleaser) builds only the standalone binaries, not this image.

## Lifecycle

- `init` (run once): `ocibnkctl init <poc_name> --no-git` with the form inputs passed as `OCIBNKCTL_*` env — creates the PoC repo at `/state/<poc_name>` with the chosen shape baked into `poc.yaml`
- `apply`: `ocibnkctl validate` then `ocibnkctl e2e --yolo --confirm-cluster <poc_name>` (cluster up → prereqs → FLO → shrink-if-tight → CNE; idempotent and resume-safe)
- `destroy`: `ocibnkctl destroy --yolo --confirm-cluster <poc_name>`

## Inputs

| Name | Type | Default | Description |
|------|------|---------|-------------|
| `poc_name` | string | `poc` | PoC/cluster name — also names the k3s node containers. **Distinct per parallel PoC on one host.** |
| `provider` | docker \| podman | `docker` | Host container runtime the k3s nodes run on |
| `tmm_nodes` | number | `1` | Worker nodes dedicated to TMM (one active TMM pod each) |
| `edge_octet` | number | `99` | 3rd octet of the bnk-edge network (192.168.\<n\>.0/24) — **distinct per cluster** or parallel PoCs collide on the same docker subnet |
| `host_profile` | standard \| small | *(unset = auto)* | Auto-detect picks `small` below the 10-core floor (sheds TMM's metrics sidecar) |
| `teems_relay` | boolean | `false` | Host-side TEEMS egress relay for hosts with lossy forwarded egress |
| `customer` | string | — | Optional label recorded in poc.yaml metadata (cosmetic) |

Inputs are seeded into `poc.yaml` via `OCIBNKCTL_*` env at `init` (ocibnkctl
v2.3.1-1+); `poc.yaml` remains the source of truth afterwards.

## Prerequisites — read before deploying

1. **F5 entitlement files must be placed in the module state before `apply`** (delivered through F5's normal channels; there is no env/flag injection for these):
   - FAR tarball → `/state/<poc_name>/keys/` (image-pull credentials for `repo.f5.com`)
   - TEEM JWT → `/state/<poc_name>/keys/.jwt`
   Run the module's `init` first (creates `/state/poc`), drop the files into the module's state volume, then deploy.
2. **Host resource floor**: ~10 cores for the full stack (`e2e` auto-runs `deploy shrink` on tighter hosts; a 4-core `host_profile: small` exists — see the upstream README).
3. **The WIDE docker socket proxy** (`docker-socket-proxy-infra`, guide §14.6): ocibnkctl builds a k3s cluster out of containers, so it needs the Docker networks/volumes/exec APIs the default narrow proxy denies. Enable it on the BNK Forge host with `COMPOSE_PROFILES=docker-infra make deploy`. Without it, `cluster up` fails at the first `docker network create` with a 403 from the proxy.

## Status

First manifest revision — validated against the BNK Forge manifest contract; a full `apply` requires F5 entitlement files and has not been exercised end-to-end from a Forge deployment yet. Expect iteration (e.g. kubeconfig reachability between the runner and the k3s server container).

## Updating

New ocibnkctl release → rebuild/publish `ghcr.io/mwiget/ocibnkctl-tools-runner` (from [`runner.Dockerfile`](https://github.com/mwiget/ocibnkctl/blob/main/runner.Dockerfile) — see [ocibnkctl `docs/RELEASE.md`](https://github.com/mwiget/ocibnkctl/blob/main/docs/RELEASE.md)), update `digest` and bump `version` in **both** `bnkforge.artifact.json` and `bnkforge.pack.json` (and any blueprint pinning this module).
