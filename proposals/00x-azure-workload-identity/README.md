# SKIP 00x - Azure Workload Identity

Summary: This SKIP proposes a method for integrating Azure Workload Identity with SpinApps, ensuring secure access to Azure services without managing secrets.

Owner: duibao55328@gmail.com (Mossaka)

Impacted Projects:

- [x] spin-operator
- [ ] `spin kube` plugin
- [ ] runtime-class-manager
- [ ] containerd-shim-spin
- [ ] Governance
- [ ] Creates a new project

Created: February 17th, 2025

Updated: N/A

## Background

Spin apps frequently interact with Azure cloud services for tasks such as storing data to CosmosDB, fetching runtime configuration variables from Key Vault, and other related tasks. Currently, authentication is managed through static credentials stored as secrets, which require manual handling and are prone to rotation.

[Microsoft Entra Workload ID](https://learn.microsoft.com/en-us/entra/workload-id/workload-identities-overview) or Workload Identity uses [Kubernetes service accounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#serviceaccount-token-volume-projection) to enable workloads to authenticate to Azure services without managing secrets using OpenID Connect (OIDC). This eliminates the need for long-lived secrets and aligns with security best practices.

This proposal is a scoped down version of the proposed [Cloud Identity and Access Management SKIP](https://github.com/spinframework/skips/pull/9) by [@devigned](https://github.com/devigned), focusing specifically on existing Azure Workload Identity. What's included in this SKIP will be discussed in the [Scope](#scope) section.

## Scope

1. Only Azure Workload Identity is supported.
2. The operator is only responsible for annotating the SpinApp resource to enable workload identity.
3. The operator is **not responsible** for creating any Azure resources.
4. The operator is **not responsible** for creating identities or federated credentials.

## Proposal

This SKIP proposes a new `workloadIdentity` field in the SpinApp resource specification.

The `workloadIdentity` field has the following structure:

- `providers`: A map of cloud providers and their configurations
  - `azure`: Configuration specific to Azure Workload Identity
    - `clientId`: The client ID of the user-assigned managed identity in Azure. Validation in the operator will be done to ensure the client ID is a valid UUID. The operator should have a clear error message if the client ID is invalid.

When this field is configured, the operator will:
1. Create a Kubernetes service account with the appropriate Azure Workload Identity annotations
2. Add the required labels for Azure Workload Identity

When this field is updated, the operator will:
1. Update the service account with the new client ID
2. Update the deployment to use the new service account

Here's an example of a SpinApp using Azure Workload Identity (some fields are omitted for brevity):
```yaml
apiVersion: core.spinkube.dev/v1alpha1
kind: SpinApp
metadata:
  name: wid-spinapp
spec:
  image: "ghcr.io/spinkube/containerd-shim-spin/examples/spin-rust-hello:v0.13.0"
  executor: containerd-shim-spin
  workloadIdentity:
    providers:
      azure:
        clientId: "YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY"
...
```

The operator will create a service account with the following annotations:

```yaml
apiVersion: core.k8s.io/v1
kind: ServiceAccount
metadata:
  name: workload-identity-sa
  annotations:
    azure.workload.identity/client-id: "YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY"
```

And the pod will have the following label (omitted for brevity):

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    azure.workload.identity/use: true
...
```

Note that before you deploy the SpinApp, you need to create the workload identity and federated credentials in Azure following the [Microsoft Entra Workload ID](https://learn.microsoft.com/en-us/azure/aks/workload-identity-deploy-cluster) documentation.

Roughly speaking, you need to create the following resources:

- An AKS cluster with the OIDC issuer and a Microsoft Entra Workload ID.
- A user-assigned managed identity
- A federated credential for the user-assigned managed identity

The Spin operator will not create or manage any of these resources. Instead, identity provisioning will be handled externally by the user. Recommended tools are:

- Terraform
- Azure CLI and ARM templates

For better developer experience, SpinKube documentation will be updated to include examples of how to provision the necessary resources using Terraform and Azure CLI.

## Alternatives considered

To quote the [Cloud Identity and Access Management SKIP](https://github.com/spinframework/skips/pull/9):

> I believe the alternative is to continue using the runtime configuration model where secrets can be stored in a secure vault. There will still need to be an initial secret to access the vault, which isn't ideal, but it is a common pattern.



