# BNK demo on native k3s (ocibnkctl)

## Solution Description

Deploy F5 BIG-IP Next for Kubernetes 2.4.0 in demo mode on a k3s cluster whose nodes run as containers on the deployment host's docker/podman runtime. No cloud account, no DPU, no SR-IOV — aimed at laptops and small lab hosts.

## What you get

- A native k3s cluster (server + TMM worker) running as containers on the deployment host
- F5 BIG-IP Next for Kubernetes 2.4.0 deployed in demo mode (FLO, cert-manager, CNE/TMM)
- Resume-safe, idempotent pipeline — re-running apply continues where it left off

## Prerequisites

- F5 FAR tarball as project secret `far_tarball` (materialized into the workspace `keys/` dir via the module's `secret_files`)
- TEEM JWT as project secret `jwt_token` (materialized as `keys/.jwt`)
- BNK Forge docker-socket proxy with container create/exec capabilities
- ~10 CPU cores and 24 GB of container-runtime memory on the deployment host for the full footprint (auto-shrink below either; ~15 GB is still needed with shrink)

## Modules

### ocibnkctl tools runner

Runs `ocibnkctl init/validate/e2e` inside the pinned `ghcr.io/mwiget/ocibnkctl-tools-runner` image; k3s nodes launch as sibling containers through the docker-socket proxy.

## Input variables

Only the customer name is required; it is recorded in the PoC repo that `ocibnkctl init` creates in the module state.
