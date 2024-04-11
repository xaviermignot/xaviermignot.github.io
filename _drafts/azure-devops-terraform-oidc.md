---
title: Federated credentials between Azure Pipelines and Terraform's Azure DevOps provider
tags: [azure, azure-devops, azure-pipelines, terraform]
img_path: /assets/img/azdo-terraform-oidc
---

Workload identity federation is becoming more and more supported in the Azure ecosystem, and there is already a lot of content on how to use it for deploying Azure resources (I briefly mention it [here]({% post_url 2023-11-27-github-runner-container-app-part1 %}#one-last-word-about-managed-identities) and [there](https://github.com/xmi-cs/aca-gh-actions-runner?tab=readme-ov-file#connect-github-with-azure) in a GitHub Actions use-case).  
Since last month it's also supported by the Terraform provider for Azure DevOps. As there are less content for this provider, let's see in this post how to configure it and make it work with a simple pipeline.

## Add a service principal user in your organization
The first thing to do is to create an _app registration_ in Entra ID (formerly Azure AD). Just give it a name and keep the other settings as default:  
![Create and App Registration in the portal](/portal-app-registration.png){: width="600"} _This can also be done with tools like the Azure CLI or PowerShell_
Then in Azure DevOps, in _Organization settings_, you can add the _service principal_ just like any normal user (don't forget to add it to at least one project):
![Add service principal in Azure DevOps](/azdo-user.png){: width="600"}

> Choosing the _Basic_ access level will use consume one licence (the first 5 users are free for each organization) 
{: .prompt-warning }

> Before doing this you need to connect your Azure DevOps organization with your Entra ID tenant, checkout the docs [here](https://learn.microsoft.com/en-us/azure/devops/organizations/accounts/connect-organization-to-azure-ad?view=azure-devops) if you need.
{: .prompt-tip }

## Create a service connection with federated credentials

## Let's add some code

## Wrapping-up
