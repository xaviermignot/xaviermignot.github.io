---
title: Today I Learned about Azure DevOps and OIDC
tags: [azure, azure-devops, azure-pipelines, terraform]
---

Over the last few weeks/months there have been lots of announcements about authentication to Azure with Open ID Connect (OIDC) and federated credentials. In short it allows to authenticate as a Service Principal without using a secret.  
There is already a lot of content on how to use this for deploying Azure resources (I briefly mention it [here]({% post_url 2023-11-27-github-runner-container-app-part1 %}#one-last-word-about-managed-identities) and [there](https://github.com/xmi-cs/aca-gh-actions-runner?tab=readme-ov-file#connect-github-with-azure) in a GitHub Actions use-case), but it also works for the Azure DevOps API... for some use-cases. In this short post I want to share what I learned about this topic.

## Authenticate to Azure DevOps using Terraform/OpenTofu

## Authenticate as a self-hosted runner
