# ocibnkctl — BNK on native k3s (host container runtime)

Container-runner module wrapping [ocibnkctl](https://github.com/mwiget/ocibnkctl) **v2.3.1-11**, which deploys F5 BIG-IP Next for Kubernetes 2.3.1 in demo mode on a k3s cluster whose nodes run **as containers on the host docker/podman runtime** — no kind/k3d, no cloud account. Aimed at low-spec hosts where a bare-metal or DPU pipeline is overkill.

## How it runs under BNK Forge

The runner image carries `ocibnkctl` plus the tools it requires (`docker` CLI, `kubectl`, `helm`, `git`). The k3s node containers are launched as **sibling containers** on the host runtime through BNK Forge's capability-scoped docker-socket proxy (`DOCKER_HOST=tcp://docker-socket-proxy:2375`, set via `state.home_env`) — the runner never mounts the raw host socket.

### Runner image provenance

The pinned image `ghcr.io/mwiget/ocibnkctl-tools-runner` (multi-arch: linux/amd64 + linux/arm64) is built and published from the upstream **ocibnkctl** repo, not from this catalog:

- **Build source:** [`runner.Dockerfile`](https://github.com/mwiget/ocibnkctl/blob/main/runner.Dockerfile) — `FROM alpine`, installs the tool set, and downloads + checksum-verifies the `ocibnkctl` binary from the matching GitHub release.
- **Who builds it:** ocibnkctl CI. The `runner-image` job in [`release.yml`](https://github.com/mwiget/ocibnkctl/blob/main/.github/workflows/release.yml) builds + pushes the multi-arch image on every `v*` tag (after goreleaser publishes the binaries) and prints the digest in the run summary. `make runner-image … PUSH=1` remains the manual/local fallback — see [`docs/RELEASE.md`](https://github.com/mwiget/ocibnkctl/blob/main/docs/RELEASE.md).

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

1. **F5 entitlement files as project secrets** (delivered through F5's normal channels). The manifest's `secret_files` block materializes them into the workspace before every run — add them to the BNK Forge project before `apply`:
   - project secret `far_tarball` → materialized at `<poc_name>/keys/f5-far-auth-key.tgz` (image-pull credentials for `repo.f5.com`)
   - project secret `jwt_token` → materialized at `<poc_name>/keys/.jwt` (TEEM activation token)
   Manually dropping the files into the module's state volume after `init` still works, but the project-secrets path survives volume recreation and is what the blueprint prerequisites describe.
2. **Host resource floor**: ~10 cores for the full stack (`e2e` auto-runs `deploy shrink` on tighter hosts; a 4-core `host_profile: small` exists — see the upstream README).
3. **The WIDE docker socket proxy** (`docker-socket-proxy-infra`, guide §14.6): ocibnkctl builds a k3s cluster out of containers, so it needs the Docker networks/volumes/exec APIs the default narrow proxy denies. Enable it on the BNK Forge host with `COMPOSE_PROFILES=docker-infra make deploy`. Without it, `cluster up` fails at the first `docker network create` with a 403 from the proxy.

## Status

Exercised end-to-end from BNK Forge deployments through the 2.3.1.x series: full `apply` (cluster up → FLO → CNE), destroy, resume, cluster auto-registration, scenario actions, and the reports viewer. Fixes discovered along the way land as new runner + module versions (see the version history in this repo's log).

## Updating

New ocibnkctl release → rebuild/publish `ghcr.io/mwiget/ocibnkctl-tools-runner` (from [`runner.Dockerfile`](https://github.com/mwiget/ocibnkctl/blob/main/runner.Dockerfile) — see [ocibnkctl `docs/RELEASE.md`](https://github.com/mwiget/ocibnkctl/blob/main/docs/RELEASE.md)), update `digest` and bump `version` in **both** `bnkforge.artifact.json` and `bnkforge.pack.json` (and any blueprint pinning this module).
