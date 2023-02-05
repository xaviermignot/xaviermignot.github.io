---
title: "Azure App Service, deployment slots (with configuration !) and Bicep"
tags: [azure, bicep, app-service]
img_path: /assets/img/bicep-app-service-slots
---

Following my [previous post]({% post_url 2022-12-24-terraform-app-service-slots %}) about blue-green deployment with Azure App Services and Terraform, I wanted to do the same thing using Bicep.  
I have started from the same demo, rewritten the IaC parts in Bicep, and adapted the GitHub Actions workflows accordingly.


## GitHub repository !

As always all code described here can be found on my [GitHub](https://github.com/xaviermignot/bicep-app-service-slots), with all the instructions in a README to run the demo by yourself.  
The demo consists in an ASP.NET web, Bicep code and GitHub Actions workflows. I will not dig too much into the App Services parts of the Bicep code in the post, you can check it in the repo if you need.


## What is blue-green deployment anyway ?

Blue-green deployment have several definitions, we will stick here to the one from [Wikipedia](https://en.wikipedia.org/wiki/Blue-green_deployment). 
Let's say we want to manage two version of our application: a _blue_ one and a _green_ one. The lifecycle of our app will follow these steps:
1. We start with the _blue_ version in production and the _green_ version in staging
2. _green_ is the future version so we work by making changes on it
3. Once the _green_ version ready we swap so it goes live and _blue_ in staging
4. We begin a new cycle of development, but this time  working on the _blue_ version as it's the new next thing
5. Once ready another swap is made,  _blue_ goes live, _green_ in staging, we are back in step 1 and start a new cycle

> Other [sources](https://www.redhat.com/en/topics/devops/what-is-blue-green-deployment) define blue-green as a gradual deployment but that's not what we will be using here.
{: .prompt-info }


## The problem with deployment slots and IaC

Using deployment slots and IaC is simple until you introduce differences between your slots at the infrastructure level. Unfortunately this happens very quick if some of your _app settings_ or your _application stack_ is managed in your IaC code.  
The problem is that the _swap_ operation uses an _imperative_ approach which conflicts with the _declarative_ approach of your IaC code. As the swap is done using the portal or the CLI, the Bicep code isn't aware of the change, so the next deployment will revert the changes and mess everything up.


## The key points of my solution

Luckily we can solve this by following these rules:  
1. The answer to the question _"which version is currently deployed in the production slot ?"_ must be stored somewhere and easily accessible
2. Every time we perform a Bicep deployment, we pass the configuration of both versions
3. We also pass to each Bicep deployment which version of the app is in production

The key point is where do we store the answer of rule n°1. In Terraform we can use the state but there is no such thing with Bicep. The trick is to use a parameter `activeApp` whose value can be either `blue` or `green`.  
Before any Bicep deployment, we can use the CLI to retrieve the `activeApp` input of the previous deployment and:
- use the other value after a swap to _save_ the new answer to _"which version is currently deployed in the production slot ?"_
- use the same value for any other deployment


## Let's see this in action !

To start at the first step of the deployment cycle described earlier, we run the [`initial-deployment`](https://github.com/xaviermignot/bicep-app-service-slots/blob/main/.github/workflows/initial-deployment.yml) workflow to provision all the resources in Azure, deploy the _blue_ version of the app in the production slot, and the _green_ version in the staging slot.

In the Bicep code, the rule n°3 (always pass which version is in production) is implemented using this parameter passed from the `main` to the `appService` module:
```
@allowed([ 'blue', 'green' ])
param activeApp string = 'blue'
```
{: file="main.bicep" }


The module that creates the App Service is call with the configuration of both version (this is rule n°2):
```
module appService 'modules/appService.bicep' = {
  name: 'deploy-app-service'
  params: {
    activeApp: activeApp
    // ...
    blueAppSettings: [
      {
        name: 'EnvironmentLabel'
        value: 'This is the blue version'
      }
    ]
    greenAppSettings: [
      {
        name: 'EnvironmentLabel'
        value: 'This is the green version'
      }      
    ]
  }
}
```
{: file="resources.bicep" }

And of course the `activeApp` parameter is used to determine which configuration is bound to the production or the staging slot:
```
resource app 'Microsoft.Web/sites@2022-03-01' = {
  // ...
  properties: {
    // ...
    siteConfig: {
      // ...
      appSettings: activeApp == 'blue' ? blueAppSettings : greenAppSettings
    }
  }
}

var slotAppSettings = activeApp == 'blue' ? greenAppSettings : blueAppSettings
resource staging 'Microsoft.Web/sites/slots@2022-03-01' = {
  // ...
  properties: {
    siteConfig: {
      // ...
      appSettings: concat(slotAppSettings, [ {
            name: 'IsStaging'
            value: true
          } ])
    }
  }
}
```
{: file="modules/appService.bicep" }

Moving forward to the third step of our deployment cycle, we make a first swap by running the [`azcli-swap`](https://github.com/xaviermignot/bicep-app-service-slots/blob/main/.github/workflows/azcli-swap.yml) workflow which uses the following command:  
```sh
az webapp deployment slot swap -g $RG_NAME -n $APP_NAME -s staging
```
This is where things would get messy if we don't follow rule n°1. The workflow does it by getting the previously deployed version, "inverting" it, and performing another deployment with the new version:
```sh
previousVersion=$(az deployment group show -g $RG_NAME -n deploy-bicep-aps-demo-resources --query properties.parameters.activeApp.value -o tsv 2>/dev/null)
[[ "$previousVersion" == 'green' ]] && versionToUse="blue" || versionToUse="green"
az deployment group create -g $RG_NAME -n deploy-bicep-aps-demo-resources -p activeApp=$versionToUse
```
Note that this deployment will have no change on the resources in Azure, it will only save the new version for the next deployments.

## Wrapping-up
