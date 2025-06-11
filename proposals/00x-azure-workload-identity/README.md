# SKIP 00x - Azure Workload Identity

Summary: This SKIP proposes a method for integrating Azure Workload Identity with SpinApps, ensuring secure access to Azure services without managing secrets.

Owner: duibao55328@gmail.com (Mossaka)

Impacted Projects:

- [x] spin-operator
- [x] `spin kube` plugin
- [ ] runtime-class-manager
- [ ] containerd-shim-spin
- [ ] Governance
- [x] Creates a new project

Created: February 17th, 2025

Updated: March 20st, 2025

## Background

Spin apps frequently interact with Azure cloud services for tasks such as storing data to CosmosDB, fetching runtime configuration variables from Key Vault, and other related tasks. Currently, authentication is managed through static credentials stored as secrets, which require manual handling and are prone to rotation.

[Microsoft Entra Workload ID](https://learn.microsoft.com/en-us/entra/workload-id/workload-identities-overview) or Workload Identity uses [Kubernetes service accounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#serviceaccount-token-volume-projection) to enable workloads to authenticate to Azure services without managing secrets using OpenID Connect (OIDC). This eliminates the need for long-lived secrets and aligns with security best practices.

This proposal is a scoped down version of the proposed [Cloud Identity and Access Management SKIP](https://github.com/spinframework/skips/pull/9) by [@devigned](https://github.com/devigned), focusing specifically on existing Azure Workload Identity. What's included in this SKIP will be discussed in the [Scope](#scope) section.

## Scope

1. The operator is only responsible for mutating the "serviceAccountName" field in the deployment to use the workload identity service account.
3. The operator is **not responsible** for creating any Azure resources.
4. The operator is **not responsible** for creating Azure Identity or Azure federated credentials.
5. `spin kube` plugin will be updated to include `serviceAccountName` as a field in the `spin kube scaffold` command.
6. `spin kube` plugin will be updated to include pod labels specifying the Azure Workload Identity is enabled.
7. A new `spin azure` plugin is proposed to create Azure resources complementing the spin operator and `spin kube` plugin.

## Proposal

This SKIP proposes three changes to the SpinKube project:
1. A new `ServiceAccountName` field in the SpinApp resource specification [spin-operator#372](https://github.com/spinframework/spin-operator/pull/372)
2. the `spin kube` plugin will be updated to include `serviceAccountName` as a field in the `spin kube scaffold` command [spin-plugin-kube#96](https://github.com/spinframework/spin-plugin-kube/pull/96) and the `spin kube` plugin will be updated to label the pod with Azure Workload Identity annotations [spin-plugin-kube#97](https://github.com/spinframework/spin-plugin-kube/pull/97)
3. A new `spin azure` plugin is proposed to create Azure resources complementing the spin operator and `spin kube` plugin [mossaka/spin-plugin-azure](https://github.com/Mossaka/spin-plugin-azure)

### 1. `ServiceAccountName` field in the SpinApp resource specification

The `serviceAccountName` field is a new field in the SpinApp resource specification. It is used to specify the name of the Kubernetes service account to use for workload identity.

Here is an example of a SpinApp using Azure Workload Identity (some fields are omitted for brevity):
```yaml
apiVersion: core.spinkube.dev/v1alpha1
kind: SpinApp
metadata:
  name: wid-spinapp
spec:
  image: "ghcr.io/spinkube/containerd-shim-spin/examples/spin-rust-hello:v0.13.0"
  executor: containerd-shim-spin
  serviceAccountName: "workload-identity-sa"
```

### 2. `spin kube` plugin

spin kube plugin will be updated to include `serviceAccountName` as a field in the `spin kube scaffold` command and a pod label with Azure Workload Identity annotations.

Here is an example of `spin kube scaffold` command output:

```bash
spin kube scaffold -f ghcr.io/squillace/cosmosworkload:0.1.0 -c azure-runtime-config.toml --azure-identity --service-account-name=workload-identity -o scaffold.yaml
```

```yaml
apiVersion: core.spinkube.dev/v1alpha1
kind: SpinApp
metadata:
  name: cosmosworkload
spec:
  image: "ghcr.io/squillace/cosmosworkload:0.1.0"
  executor: containerd-shim-spin
  replicas: 2
  podLabels:
    azure.workload.identity/use: "true"
  serviceAccountName: workload-identity
  runtimeConfig:
    loadFromSecret: cosmosworkload-runtime-config
---
# secrete is omitted for brevity
```

The noticable changes are:
1. the new `azure.workload.identity/use` label is added to the pod if the flag `--azure-identity` is provided to the `spin kube scaffold` command.
2. the `serviceAccountName` field is added to the SpinApp resource specification if the flag `--azure-identity` is provided to the `spin kube scaffold` command.

### 3. `spin azure` plugin

This SKIP proposes a new Spin plugin for a full experience of deploying and managing Spin applications on Azure Kubernetes Service (AKS) with workload identity. This plugin is primarily focused on the following:


The plugin offers a comprehensive set of commands to 

1. Login to Azure
2. Create a new AKS cluster with workload identity enabled and Spin Operator installed
3. Create a new identity with Kubernetes service account and federated credential
4. Assign necessary roles to access Azure services
5. Deploy a Spin application to your cluster

This plugin complements the spin operator and `spin kube` plugin by providing a seamless workflow for Azure-specific resources and configuration:

1. Use `spin kube scaffold` to generate a SpinApp manifest with workload identity configured
2. Use `spin azure` to create and configure all necessary Azure infrastructure
3. The spin operator handles the Kubernetes deployment with proper identity configuration

The plugin handles the Azure-specific aspects that the spin operator and `spin kube` plugin do not cover, thus providing a complete end-to-end experience for deploying Spin applications on AKS with workload identity.

Here is an example of `spin azure` help page:


```console
$ spin azure --help                                                                                                          
A CLI tool for managing Spin apps on Azure Kubernetes Service (AKS).
This tool helps you create and manage AKS clusters with workload identities enabled,
and deploy Spin apps to them.

Usage:
  azure [command]

Examples:
  # Login to Azure
  spin azure login

  # Create a new AKS cluster
  spin azure cluster create --name my-cluster --resource-group my-rg --location eastus

  # Use an existing AKS cluster
  spin azure cluster use --name existing-cluster --resource-group existing-rg

  # Create a new identity
  spin azure identity create --name my-custom-identity

  # Create an identity without a Kubernetes cluster
  spin azure identity create --name my-identity --resource-group my-rg --skip-service-account

  # Use an existing identity and create a service account
  spin azure identity use --name my-custom-identity --create-service-account

  # Assign Azure CosmosDB role to an identity
  spin azure assign-role cosmosdb --name my-cosmos --resource-group my-rg

  # Deploy a Spin application
  spin azure deploy --from path/to/spinapp.yaml
 
  # Output the config
  spin azure config show

  # Reset the config
  spin azure config reset -y

Available Commands:
  assign-role Assign Azure RBAC roles to managed identities
  cluster     Manage AKS clusters for Spin applications
  completion  Generate the autocompletion script for the specified shell
  config      Manage configuration
  deploy      Deploy Spin applications to AKS
  help        Help about any command
  identity    Manage Azure managed identities for Spin workloads
  login       Log in to Azure
  logout      Log out from Azure

Flags:
  -h, --help   help for azure

Use "azure [command] --help" for more information about a command.
```

To use the plugin, you need to install it first:

```console
$ spin plugins update
$ spin plugins install azure
```

> Note: the above example is just for illustration. The plugin you installed is likely to be different.

## Alternatives considered

The first question to ask is whether the `spin azure` plugin is necessary and can it be part of the spin operator. An early plan was to include the Azure pod labels and sericeAccountName in the SpinApp resource specification. When the SpinApp is created, the operator will create the Azure resources and configure the pod with the Azure Workload Identity annotations.

The reason why we don't want to do that is two folds:
1. The spin operator should remain provider agnostic. It should not be concerned with the underlying cloud provider - Azure or otherwise.
2. More importantly, there are security concerns with the operator being able to create Azure Managed Identities and Federated Credentials.

Thus, we need to keep the changes to the operator minimal and move the Azure-specific changes to a Spin plugins.

The `spin kube` plugin might be a good candidate, but as the name suggests, it is a plugin for interacting with Kubernetes resources. It is not a good candidate for interacting with Azure resources. It is clear to me that a Azure-specific plugin is necessary to manage Azure resources, such as Authenticating with Azure, creating AKS clusters, creating managed identities, and assigning roles.

The `spin kube scaffold` command, however, is still useful for generating the SpinApp that has the necessary Azure labels and serviceAccountName. I have considered the alternative of having `spin kube` remain provider agnostic and have `spin azure` to be responsible for generating a SpinApp that has the necessary Azure labels. I didn't go with that because I didn't have time to implement it. But if folks think that's a good idea, I can give it a try.

Below is the original proposal for the `workloadIdentity` field in the SpinApp resource specification, which I rejected due to the security concerns mentioned above:

The `workloadIdentity` field has the following structure:

- `workloadIdentity`: Configuration for Workload Identity
  - `serviceAccountName`: The name of the Kubernetes service account to use for workload identity.
  - `providerMetadata`: A map of cloud provider-specific configuration.

The Azure provider metadata has the following structure:

- `azure`: true | false

When this field is configured, the operator will:
1. Mutate the deployment to use the workload identity service account
2. Label the pod with Azure Workload Identity annotations

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
    serviceAccountName: "workload-identity-sa"
    providerMetadata:
      azure: true
```


And the pod will have the following label and service account name (omitted for brevity):

```yaml
Name: pod-name
ServiceAccount: workload-identity-sa
Labels:
  azure.workload.identity/use: true
...
```
