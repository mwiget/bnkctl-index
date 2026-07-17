# roksbnkctl — BNK on IBM Cloud ROKS/IKS

Container-runner module wrapping [roksbnkctl](https://github.com/jgruberf5/roksbnkctl) **v1.20.0**, which provisions a ROKS/IKS cluster on IBM Cloud and brings up the F5 BIG-IP Next for Kubernetes (BNK) stack. The tool embeds its own Terraform modules and internalized `kubectl`/`oc`/`ibmcloud` tooling, so the digest-pinned `roksbnkctl-tools-runner` image (published per release by upstream) is fully self-contained.

## Lifecycle

- `init` (run once): `roksbnkctl init --non-interactive` — builds the workspace `config.yaml` entirely from `ROKSBNKCTL_*` env vars (region, resource group, prefix, cluster name/create), which the manifest maps from the module inputs. The IBM Cloud API key arrives via BNK Forge's `ibmcloud` credential injection (`IBMCLOUD_API_KEY`).
- `apply`:
  - `cluster up --auto` — gated by the `provision_cluster` input; set false to attach to an existing cluster
  - `bnk up --auto` — gated by the `deploy_bnk` input
- `destroy`: `roksbnkctl down --auto` — terraform-destroys the workspace (BNK, and the cluster if this workspace provisioned it)

Workspace state persists across steps in the module's `/state` volume (`ROKSBNKCTL_HOME=/state`).

## Inputs

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `region` | string | yes | IBM Cloud region (e.g. `us-south`) |
| `resource_group` | string | yes (default `default`) | IBM Cloud resource group |
| `prefix` | string | yes | Resource-name prefix for everything the workspace creates |
| `cluster_name` | string | yes | New cluster name, or existing cluster name/ID when attaching |
| `provision_cluster` | boolean | no (default `true`) | Provision a new ROKS cluster, or attach to an existing one |
| `deploy_bnk` | boolean | no (default `true`) | Deploy BNK after the cluster phase |

## Credentials

Requires an `ibmcloud` credential configured in BNK Forge (IBM Cloud API key). BNK entitlement (FAR archive / subscription JWT) per the roksbnkctl documentation.

## Updating

Upstream publishes a `roksbnkctl-tools-runner` image per release (`ghcr.io/jgruberf5/roksbnkctl-tools-runner:vX.Y.Z`). To move to a new release: resolve the new tag's digest (`docker buildx imagetools inspect …:vX.Y.Z`), verify the CLI surface used in `steps` still exists (`docker run --rm <image> --help`), then update `digest` and bump `version` in **both** `bnkforge.artifact.json` and `bnkforge.pack.json`, and bump any blueprint pinning this module (blueprint releases are immutable — bump `blueprint.version`).
