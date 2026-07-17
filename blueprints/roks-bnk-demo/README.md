# BNK on IBM Cloud ROKS (roksbnkctl)

## Solution Description

Provision a Red Hat OpenShift (ROKS) or IKS cluster on IBM Cloud and deploy F5 BIG-IP Next for Kubernetes using the roksbnkctl tools-runner container. Everything runs inside a digest-pinned image — no host tooling required.

## What you get

- A ROKS/IKS cluster provisioned in the selected IBM Cloud region (optional — an existing cluster can be reused)
- F5 BIG-IP Next for Kubernetes installed and running on the cluster
- Deployment outputs captured in module state for follow-on modules

## Prerequisites

- IBM Cloud API key available as an ibmcloud credential in BNK Forge
- BNK entitlement (FAR archive / subscription JWT) per the roksbnkctl documentation

## Modules

### roksbnkctl tools runner

Runs `roksbnkctl cluster up` and `roksbnkctl bnk up` inside the pinned `ghcr.io/jgruberf5/roksbnkctl-tools-runner` image, persisting state in the module's `/state` volume.

## Input variables

Set the IBM Cloud region and cluster name. Toggle `provision_cluster` off to reuse an existing cluster, or `deploy_bnk` off to stop after cluster provisioning.
