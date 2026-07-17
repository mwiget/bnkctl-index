# bnkctl-index — ctl tools catalog for BNK Forge

A curated index of `*bnkctl`-style CLI tools packaged as [BNK Forge](https://github.com/f5devcentral/bnk-forge) container-runner modules. Register this one repo as a catalog source in BNK Forge and every tool listed here becomes discoverable and importable from the Catalog UI — no host binaries, no per-tool setup.

## Tools

| Tool | What it deploys | Runner image | Upstream |
|------|-----------------|--------------|----------|
| [roksbnkctl](tools/roksbnkctl/) | F5 BIG-IP Next for Kubernetes on IBM Cloud ROKS/IKS | `ghcr.io/jgruberf5/roksbnkctl-tools-runner` | [jgruberf5/roksbnkctl](https://github.com/jgruberf5/roksbnkctl) |

## Blueprints

| Blueprint | Description |
|-----------|-------------|
| [roks-bnk-demo](blueprints/roks-bnk-demo/) | Provision a ROKS/IKS cluster and deploy BNK using roksbnkctl |

## Using this index in BNK Forge

1. In the BNK Forge UI, open **Catalog** and register this repository's URL as a source (blueprint or module source — either works; the dual-content sync discovers both the `tools/` modules and the `blueprints/`).
2. **Sync** the source. Discovered modules land in the module library; blueprint releases appear as `discovered`.
3. **Import** the blueprint release(s) you want. Imported blueprints are deployable as projects from the Catalog.

Everything runs inside digest-pinned container images pulled at deploy time — nothing from this repo executes on the Forge host, and no CLI binaries need to be installed anywhere.

## Repo contract

- `tools/<name>/` — one directory per tool, containing:
  - `bnkforge.pack.json` — catalog identity. `module.path` **must equal** the directory path (e.g. `tools/roksbnkctl`); the sync rejects mismatches.
  - `bnkforge.artifact.json` — runtime contract: digest-pinned `container_image` (registry host must be on the Forge allowlist: `ghcr.io`, `quay.io`, `docker.io`, `registry.k8s.io`), argv-only `steps` (no shell), `/state` mount for persistence.
- `blueprints/<name>/` — one directory per blueprint, containing `forge-blueprint.json` (+ optional `README.md`). Module references use the catalog path (`tools/<name>`) and a **pinned** version — `latest` is rejected.
- **Releases are immutable.** BNK Forge hashes synced content; changing a blueprint without bumping `blueprint.version` produces a sync conflict, not an update. Same discipline applies to artifact digests: new image → new module/artifact version.

## Adding a tool

1. Publish a runner image for the tool (typically `FROM gcr.io/distroless/static` + `COPY <ctl> /usr/local/bin/`), multi-arch, to an allowlisted registry. Record the manifest **digest**.
2. Copy `tools/roksbnkctl/` as a template. Set `module.path` to the new directory, pin the image digest, and describe the tool's lifecycle as `steps` (argv arrays only).
3. Optionally add a blueprint under `blueprints/` composing the module with a guided input form.
4. Validate against a bnk-forge checkout before opening a PR:

```bash
cd <bnk-forge>/backend && python3 - <<'EOF'
import json
from services.module_metadata import ModuleMetadataValidator
pack = json.load(open("<bnkctl-index>/tools/<name>/bnkforge.pack.json"))
art  = json.load(open("<bnkctl-index>/tools/<name>/bnkforge.artifact.json"))
v = ModuleMetadataValidator()
v.validate_pack_manifest(pack)
v.validate_artifact_manifest(art, registry_host_allowlist=["ghcr.io", "quay.io", "docker.io", "registry.k8s.io"])
print("OK")
EOF
```
