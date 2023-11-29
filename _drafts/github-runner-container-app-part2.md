---
title: "Scale your org's GitHub runners in Azure Container Apps"
tags: [github-actions, azure, container-apps, bicep]
img_path: /assets/img/gh-runner-aca-part-1
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

## Updates on the Bicep code

### Choosing the right version of the scaler


## About ephemeral runners

## Testing the runner at scale

## Wrapping-up