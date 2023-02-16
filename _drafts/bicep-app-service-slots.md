---
title: "Azure App Service, deployment slots (with configuration !) and Bicep"
tags: [azure, bicep, app-service]
img_path: /assets/img/bicep-app-service-slots
image:
  path: banner.png
---

This is the second and last post of the series about deploying Azure App Services with Bicep. In the [first episode]({% post_url 2023-02-09-app-service-bicep-github-actions %}), we have created an App Service with Bicep, and deployed some code to it, all with the help of GitHub Actions.  
This time we will add blue-green deployment on top of that, using App Service deployment slots. Why shall we do that ? Because it will make our upcoming deployment smoother, without any downtime. Let's go !


## What is blue-green deployment anyway ?

Blue-green deployment has several definitions, we will stick here to the one from [Wikipedia](https://en.wikipedia.org/wiki/Blue-green_deployment). 
Let's say we want to manage two version of our application: a _blue_ one and a _green_ one. The lifecycle of our app will follow these steps:
1. We start with the _blue_ version in production and the _green_ version in staging
2. _green_ is the future version so we work by making changes on it
3. Once the _green_ version ready we swap so it goes live and _blue_ moves to staging
4. We begin a new cycle of development, but this time working on the _blue_ version as it becomes the next version
5. Once ready another swap is made, _blue_ goes live, _green_ moves to staging, we are back in step 1 and we start over

> Other [sources](https://www.redhat.com/en/topics/devops/what-is-blue-green-deployment) define blue-green as a gradual deployment but that's not what we will be using here.
{: .prompt-info }


## The problem with deployment slots and IaC

> Deployment slots is a feature of Azure App Services, you can learn more about it [here](https://learn.microsoft.com/en-us/azure/app-service/deploy-staging-slots) if you're not already familiar with it.
{: .prompt-info }

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


## GitHub repository !

As always all code described here can be found on my [GitHub](https://github.com/xaviermignot/bicep-app-service-slots), with all the instructions in a README to run the demo by yourself.  
The demo consists in an ASP.NET web, Bicep code and GitHub Actions workflows. 


## Let's see this in action !

### First deployment
To start at the first step of the deployment cycle described earlier, we run the [`initial-deployment`](https://github.com/xaviermignot/bicep-app-service-slots/blob/main/.github/workflows/initial-deployment.yml) workflow to provision all the resources in Azure, deploy the _blue_ version of the app in the production slot, and the _green_ version in the staging slot.

This is what we have in production:
![The blue app in production](/01-blue-app-prod.png)

And in staging:
![The green app in staging](/02-green-app-staging.png)

In the Bicep code, the rule n°3 (always pass which version is in production) is implemented using this parameter passed from the `main` module to the `appService` module:
```
@allowed([ 'blue', 'green' ])
param activeApp string = 'blue'
```
{: file="infra/main.bicep" }

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
{: file="infra/resources.bicep" }

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

resource staging 'Microsoft.Web/sites/slots@2022-03-01' = {
  // ...
  properties: {
    siteConfig: {
      // ...
      appSettings: activeApp == 'blue' ? greenAppSettings : blueAppSettings
    }
  }
}
```
{: file="infra/modules/appService.bicep" }

Lastly the `activeApp` parameter is bound to the Bicep deployment, so it can be retrieved easily and is even visible in the Azure portal:
![Blue activeApp in the Azure portal](/03-deployment-blue.png)_This help us to implement the rule n°1: save which version is live somewhere_

### Swapping version
Moving forward to the third step of our deployment cycle, we make a first swap by running the [`azcli-swap`](https://github.com/xaviermignot/bicep-app-service-slots/blob/main/.github/workflows/azcli-swap.yml) workflow which uses the following command:  
```sh
az webapp deployment slot swap -g $RG_NAME -n $APP_NAME -s staging
```
{: file=".github/workflows/azcli-swap.yml" }
As expected the `swap` command puts the green version in production:
![The green app in production](/04-green-app-prod.png)
And the blue one is now in staging:
![The blue app in staging](/05-blue-app-staging.png)
Everything looks good but things would get messy if the workflow stopped here, as it still has to follow rule n°1:  "always save which version is live somewhere".  
This is done in two steps, first the workflow gets the previous `activeApp` value and "inverts" it using Bash:
```sh
# Get the activeApp parameter of the latest deployment
previousVersion=$(az deployment group show -g $RG_NAME -n deploy-bicep-aps-demo-resources --query properties.parameters.activeApp.value -o tsv 2>/dev/null)
# "Inverts" the value: blue => green, green => blue
[[ "$previousVersion" == 'green' ]] && versionToUse="blue" || versionToUse="green"
# Outputs the inverted value for next job
echo "appVersion=$versionToUse" >> $GITHUB_OUTPUT
```
{: file=".github/workflows/azcli-swap.yml" }
Then another deployment has to be made in order to save `green` as the new `activeApp` value:
{% raw %}
```yaml
update-active-app:
  uses: ./.github/workflows/bicep-deploy.yml
  needs: swap
  with:
    activeApp: ${{ needs.swap.outputs.appVersion }}
    updateSecrets: false
  secrets: inherit
```
{: file=".github/workflows/azcli-swap.yml" }
{% endraw %}
Note that this deployment will have no change on the resources in Azure, it will only save the new version for the next deployments:
![Green activeApp in the Azure portal](/06-deployment-green.png)

### What about next deployments ?
Now that our first swap has been made, we are at the step 4 or the deployment cycle described [earlier](#what-is-blue-green-deployment-anyway-) in this post.  
What could happen next ? In case of a rollback, if we run the `azcli-swap` workflow again, the green app will be put back in staging, and the green one in production. That's an handy benefit of using deployment slots to rollback easily.  
If any change has to be done on the infrastructure _without_ swapping, we just have to get the `activeApp` value of the latest Bicep deployment, and use the same value.  


## Wrapping-up

This marks the end of this post and this series. I hope this gives you a baseline on how to manage your App Services with Bicep and GitHub Actions.  
A always feel free to reach out if you have any question, you can even drop a comment below (this is new). Thanks for reading and happy learning 🤓