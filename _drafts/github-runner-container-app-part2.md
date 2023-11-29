---
title: "Scale your org's GitHub runners in Azure Container Apps"
tags: [github-actions, azure, container-apps, bicep]
img_path: /assets/img/gh-runner-aca-part-2
image:
  path: banner.png
---

This is the second post of the series about hosting your GitHub org's self-hosted runners in Azure Container Apps. The [first episode]({% post_url 2023-11-27-github-runner-container-app-part1 %}) covers the essentials of the solution, concluding with a testing workflow running in an Azure Container App.  
This post will focus on _scaling_ the self-hosted runners, using the builtin KEDA integration of Azure Container Apps.

## In the previous episode...
If you haven't my previous post you can check out [this section]({% post_url 2023-11-27-github-runner-container-app-part1 %}#github-repository-and-organization) that presents the solution used in this series.  
> There is also a GitHub [repository](https://github.com/xmi-cs/aca-gh-actions-runner) with all the code and instructions on how to deploy it in your environment.
{: .prompt-info }

## The KEDA GitHub runner scaler
[KEDA](https://keda.sh/) is the de-facto autoscaler of the Kubernetes ecosystem. It can sale workloads in a cluster based on event-driven sources called _scalers_. Among the many available scalers there is the [GitHub Runner Scaler](https://keda.sh/docs/latest/scalers/github-runner/), and like any KEDA scaler, we can use it to scale our Container App.  
It works by querying the GitHub REST API to get the number of queued jobs, and will add or remove _replicas_ to the Container App accordingly. This means we need to configure how the scaler will authenticate against the GitHub REST API, and which filters to count the jobs that are worth a scale. 

## Adding the scaler in the Bicep code
Before jumping into some Bicep code, let me save you some time by telling a little trap I have fallen into while preparing this post.

### Choosing the right version of the scaler
The link to the KEDA documentation in the previous paragraph leads to the latest version of KEDA (`2.12` at the time of writing this post). Before browsing the docs I strongly advise you to check the KEDA version used by your Container Apps Environment:
![KEDA version in the Azure portal](/02-portal-environment.png)_Looks like I don't have the latest version 🥹_

Azure Container Apps is a _managed_ service where you don't manage the KEDA version used by the underneath cluster. Using Azure Kubernetes Service will probably give you more control on this, but with Container Apps the updates are rolled by service.  

Before I understood this, I was trying to use the scaler's authentication with a GitHub App private key, which is not available in the KEDA version used by my Container App Environment. Once we know that, let's see how to include the scaler in the Bicep code using the right version of the docs.

### Adding a scale rule

## About ephemeral runners

## Testing the runner at scale

## Wrapping-up