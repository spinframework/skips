# SKIP 004 - SpinKube Google Cloud

Summary: A repository for all things Google Cloud related in the SpinKube project.

Owners: Kedaar Sridhar <kedaar.sridhar@fermyon.com>, Nick Eberts <nickeberts@google.com>, Gari Singh <garisingh@google.com>

Impacted Projects:

- [ ] spin-operator
- [ ] `spin kube` plugin
- [ ] runtime-class-manager
- [ ] containerd-shim-spin
- [ ] Governance
- [x] Creates a new project

Created: July 22nd, 2024

## Background

Following the success of integrating SpinKube with Azure Kubernetes Service (AKS) through [SKIP 002](https://github.com/spinkube/skips/tree/main/proposals/002-skips), which resulted in the creation of an Azure-specific repository, we aim to bolster our integration efforts by expanding to Google Cloud and Google Kubernetes Engine (GKE).

After discussions with the GKE team, we have identified a strong interest in pursuing this integration to provide seamless deployment experiences for users on Google Cloud. This initiative will help increase SpinKube's visibility, usability, and adoption within the Google Cloud 

## Proposal

Inspired by the successful implementation of [SKIP 002](https://github.com/spinkube/skips/tree/main/proposals/002-skips) for Azure, we propose to create a new repository under the SpinKube organization for Google Cloud called `gcp` to host any Google Cloud-specific configurations and integrations.

The `gcp` repository will contain the basic Helm chart for deploying SpinKube on GKE clusters. Additionally, it will include a marketplace-specific Helm chart for a future Google Cloud marketplace listing. The owners/maintainers of this repository will be Nick Eberts and Gari Singh from Google, along with Fermyon engineering support for this initial integration.

This repository will serve as the central hub for all Google Cloud-related work within the SpinKube project, making it easier for the community to contribute and collaborate.
