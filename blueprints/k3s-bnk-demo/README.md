# BNK demo on native k3s (ocibnkctl)

## Solution Description

Deploy F5 BIG-IP Next for Kubernetes 2.3.1 in demo mode on a k3s cluster whose nodes run as containers on the deployment host's docker/podman runtime. No cloud account, no DPU, no SR-IOV — aimed at laptops and small lab hosts.

## What you get

- A native k3s cluster (server + TMM worker) running as containers on the deployment host
- F5 BIG-IP Next for Kubernetes 2.3.1 deployed in demo mode (FLO, cert-manager, CNE/TMM)
- Resume-safe, idempotent pipeline — re-running apply continues where it left off

## Prerequisites

- F5 FAR tarball in the module state at `/state/poc/keys/` (after init, before deploy)
- TEEM JWT in the module state at `/state/poc/keys/.jwt`
- BNK Forge docker-socket proxy with container create/exec capabilities
- ~10 CPU cores on the deployment host (auto-shrink on tighter hosts)

## Modules

### ocibnkctl tools runner

Runs `ocibnkctl init/validate/e2e` inside the pinned `ghcr.io/mwiget/ocibnkctl-tools-runner` image; k3s nodes launch as sibling containers through the docker-socket proxy.

## Input variables

Only the customer name is required; it is recorded in the PoC repo that `ocibnkctl init` creates in the module state.
