# roksbnkctl — BNK on IBM Cloud ROKS/IKS

Container-runner module wrapping [roksbnkctl](https://github.com/jgruberf5/roksbnkctl), which provisions a ROKS/IKS cluster on IBM Cloud and brings up the F5 BIG-IP Next for Kubernetes (BNK) stack. The tool embeds its own Terraform modules and internalized `kubectl`/`oc`/`ibmcloud` tooling, so the digest-pinned `roksbnkctl-tools-runner` image is fully self-contained.

## Lifecycle

- `init` (run once): `roksbnkctl init --home /state`
- `apply`:
  - `cluster up` — gated by the `provision_cluster` input; skip to reuse an existing cluster
  - `bnk up` — gated by the `deploy_bnk` input; writes outputs to `/state/outputs.json`

State persists across steps in the module's `/state` volume (`ROKSBNKCTL_HOME=/state`).

## Inputs

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `region` | string | yes | IBM Cloud region (e.g. `us-south`) |
| `cluster_name` | string | yes | ROKS/IKS cluster to create or target |
| `provision_cluster` | boolean | no (default `true`) | Create the cluster, or reuse an existing one |
| `deploy_bnk` | boolean | no (default `true`) | Deploy BNK after the cluster is up |

## Credentials

Requires an `ibmcloud` credential configured in BNK Forge (IBM Cloud API key). BNK entitlement (FAR archive / subscription JWT) per the roksbnkctl documentation.

## Updating

New roksbnkctl release → publish/locate the new `roksbnkctl-tools-runner` image, update the `digest` and bump `version` in **both** `bnkforge.artifact.json` and `bnkforge.pack.json` (and any blueprint pinning this module).
