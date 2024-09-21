---
title: "Azure Pipelines: How to get the current service principal id"
description: How to get the id of the service principal running the current task in Azure Pipelines
tags: [azure-devops, azure-pipelines]
media_subpath: /assets/img/azdo-pipeline-sp-id
image:
  path: banner.webp
  alt: Image by Freepik
date: 2024-06-23 22:00:00
---

This a quick post for sharing a solution to a problem I have faced recently: in an Azure pipeline running Azure CLI tasks, how can I get the current service principal id ?  
Well, it's a little less simple than it sounds, let's find out !

## The solution
I will go straight to the point here and reveal the solution with this command:  
`az ad sp show --id "$(az account show --query user.name -o tsv)" --query id -o tsv`

It can be used like this in your pipeline's yaml file:
```yaml
steps:
  - task: AzureCLI@2
    name: GetCurrentSpId
    inputs:
      azureSubscription: your-azurerm-service-connection
      scriptType: bash
      scriptLocation: inlineScript
      inlineScript: |
        az ad sp show --id "$(az account show --query user.name -o tsv)" --query id -o tsv
```
{: file=".azuredevops/azure-pipeline.yml" }

So you can take this and adapt it to you situation right now, that's fine, I do this all the time when I stumble upon some random blog post 🤫  
If you want to know _why_ this works, you can also read on 😉

## A few explanations
Initially I had this question because I needed to access a Key Vault's existing secret in a Bicep deployment. That implies creating a role assignment with the _Secrets User_ role to the service principal running my pipeline (and of course the Bicep deployment).  

First I tried to get the sp id with the `az ad signed-in-user show` command but it works only for users and returns an error (`/me request is only valid with delegated authentication flow.`) when authenticated as a service principal.  

Another approach is to use the `addSpnToEnvironment` input of the `AzureCLI` task: it adds the `$servicePrincipalId` to the script but using this id for role assignment doesn't work out of the box.  
As confusing as things can be in the world of AzureAD/Entra ID, the `$servicePrincipalId` variable exposes the _Application ID_ of your _Enterprise Application_, but for a role assignment you need the _Object ID_:
![Enterprise App's UI in Azure Portal](/enterprise-app-ids.webp){: width="400"}_You can check if you use the right id here in the Azure Portal_

Obviously if the wrong id is used for the role assignment, it will look like this in the portal:
![Identity not found for role assignment in Azure Portal](/identity-not-found.webp)_This is not fine_

So finally to make this work we need to use the `az ad sp show` command to get the _Object ID_ from the _Application ID_. It still can be done like this with the `addSpnToEnvironment` input:
```yaml
steps:
  - task: AzureCLI@2
    name: GetCurrentSpId
    inputs:
      addSpnTEnvironment: true
      azureSubscription: your-azurerm-service-connection
      scriptType: bash
      scriptLocation: inlineScript
      inlineScript: |
        az ad sp show --id "$servicePrincipalId" --query id -o tsv
```
{: file=".azuredevops/azure-pipeline.yml" }

Or without the `addSpnToEnvironment` input by combining the `az account show` command (to get the _Application ID_) with the `az ad sp show` command as shown at the beginning of this post.  
With any of these two approaches the role assignment will appear like this in the portal:
![Identity found for the role assignment in Azure Portal](/identity-found.webp)_This is much better_

## Wrapping-up
In this post we have seen that getting the current service principal of an Azure Pipeline is not that easy, but not that complicated either. I hope it's useful, I'm pretty sure my future self will be happy to check this post in a few months or years 😅