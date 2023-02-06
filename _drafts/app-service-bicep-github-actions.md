---
title: "Deploy App Services with Bicep & GitHub Actions"
tags: [azure, bicep, app-service, github-actions]
img_path: /assets/img/bicep-app-service-slots
---

Following my [previous post]({% post_url 2022-12-24-terraform-app-service-slots %}) about blue-green deployment with Azure App Services and Terraform, I wanted to do the same thing using Bicep. 
I have started from the same demo, rewritten the IaC parts in Bicep, and adapted the GitHub Actions workflows accordingly.
This time I will break this in a series of two posts. This is the first one about how to provision the App Services in Bicep and deploy some code to it, using GitHub Actions workflows.  
The second post will focus on the deployment slots and blue-green deployment stuff.


## GitHub repository !

As always all code described here can be found on my [GitHub](https://github.com/xaviermignot/bicep-app-service-slots), with all the instructions in a README to run the demo by yourself.  
The demo consists in an ASP.NET web, Bicep code and GitHub Actions workflows.  
The repository contains the materials of both posts of the series, in this post I will focus on provisioning a simple Web App (without slots) and on the GitHub Actions parts.


## Creating the Azure resources with Bicep

Before deploying any code we need to create two resources:
- An App Service Plan who represents the managed pool of machine(s) running our app
- An App Service which is the logical representation of our app, running in the plan

Here is the code for the App Service Plan:
```
param location string
param project string

resource plan 'Microsoft.Web/serverfarms@2022-03-01' = {
  name: 'asp-${project}'
  location: location

  kind: 'app,linux'

  properties: {
    reserved: true
  }

  sku: {
    name: 'S1' // We use Standard S1 as we'll need deployment slots later
  }
}

output planId string = plan.id // The resource id of the plan will be needed for the App Service
```
{: file=infra/modules/appServicePlan.bicep }

Note the few gotchas related to the use of a Linux plan: we must use `app,linux` for the `kind` and set the `reserved` property to `true`.

> Every Bicep module contains at least the following parameters:
- `project` contains a suffix used in the naming of all resources (following the [guidelines](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations) for the prefixes)
- `location` contains the Azure region also used for all resources
{: .prompt-info }

For the App Service, the code is simple but there is another gotcha with the application _stack_, i.e. which platform/language our app will be built with, .NET, Python, Go, etc.     
For Linux apps we have to use the `LinuxFxVersion`, the list of possible values can be retrieved from the CLI using the command `az webapp list-runtimes --os linux`.  
But be careful, even if this command outputs values with colons (like `DOTNETCORE:6.0`), these must be replaced by pipes (like `DOTNETCORE|6.0`), otherwise it won't work.  

Apart from that the code is simple, here is the full resource:
```
param location string
param project string

param appName string
param planId string

resource app 'Microsoft.Web/sites@2022-03-01' = {
  name: 'app-${project}-${appName}'
  location: location

  properties: {
    serverFarmId: planId // This is were we use the resource id of the plan from the previous module
    reserved: true

    siteConfig: {
      linuxFxVersion: 'DOTNETCORE|6.0' // Here is the application stack gotcha
    }
  }
}

output appServiceName string = app.name
```
{: file=infra/modules/appService.bicep }

The Bicep code is now ready so we will see how to deploy it using GitHub Actions.

## The GitHub Actions workflows

## Wrapping-up


