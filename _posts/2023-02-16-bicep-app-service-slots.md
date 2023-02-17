---
title: "App Services with Bicep & GitHub Actions, part 2: Deployments slots"
tags: [azure, bicep, app-service, github-actions]
img_path: /assets/img/bicep-app-service-slots
image:
  path: banner.png
date: 2023-02-16 23:00:00
---

This is the second and last post of the series about deploying Azure App Services with Bicep. In the [first episode]({% post_url 2023-02-09-app-service-bicep-github-actions %}), we have created an App Service with Bicep, and deployed some code to it, all with the help of GitHub Actions.  
This time we will add blue-green deployment on top of that, using App Service deployment slots. Why shall we do that ? Because it will make our upcoming deployment smoother, without any downtime.


## What is blue-green deployment anyway ?

Blue-green deployment has several definitions, we will stick here to the one from [Wikipedia](https://en.wikipedia.org/wiki/Blue-green_deployment):
> In software engineering, blue-green (also blue/green) deployment is a method of installing changes to a web, app, or database server by swapping alternating production and staging servers.  

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

The key point is where do we store the answer of rule n°1. In Terraform we can use the state but there is no such thing with Bicep.  
The trick is to use a deployment parameter `activeApp` whose value can be either `blue` or `green`. Then we can use the CLI to retrieve the parameter value of the latest deployment. This will be demoed in a moment.


## GitHub repository !

As always all code described here can be found on my [GitHub](https://github.com/xaviermignot/bicep-app-service-slots), with all the instructions in a README to run the demo by yourself.  
The demo consists in an ASP.NET web app, Bicep code and GitHub Actions workflows. In this post I will focus on the changes made from the samples of the first post of the series. To see the full picture you can browse or clone the repository.


## Some screenshots of the demo

Let's see what we are going to build, starting at the first step of our deployment cycle. In the previous post we already had the _blue_ version of the app in production:
![The blue app in production](/01-blue-app-prod.png)

This time we are going to add the _green_ version in the _staging_ slot:
![The green app in staging](/02-green-app-staging.png)_Notice the URL with the -staging suffix in the subdomain_


## Updates on the Bicep code

Let's see what we need to change in the Bicep code to make this happen.

### Add a deployment slot
Adding a deployment slot is simple, we give it a name and set the App Service resource as the parent:
```
resource staging 'Microsoft.Web/sites/slots@2022-03-01' = {
  name: 'staging'
  location: location
  parent: app // This is a symbolic reference to the App Service resource

  properties: {
    siteConfig: {
      linuxFxVersion: linuxFxVersion
    }
  }
}
```
{: file="infra/modules/appService.bicep" }
We also set the same location and application stack as the parent App Service.

### Add the `activeApp` parameter
Here we are starting to follow the rules established earlier, starting with rule n°3: always pass which value is active in production. This is done with the `activeApp` parameter, declare in the `main` module and passed to the module in charge of the App Service creation:
```
@allowed([ 'blue', 'green' ])
param activeApp string = 'blue'
```
{: file="infra/main.bicep" }

### Manage the blue and the green configuration
After rule n°3 we have to follow rule n°2: pass the configuration of all versions at each deployment.  
One of the key point of the demo is to have different values in the _app settings_ of the production and the staging slot. When the `appService` module is called, we pass two arrays of app settings, for the blue app and for the green app:
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
Note that we also pass the `activeApp` parameter as mentioned earlier.  
In the `appService` module, this parameter is used to determine which configuration is bound to the production or the staging slot:
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


### Add a sticky setting
Notice the red panel only visible in the staging slot ? It's managed using a slot or sticky setting, i.e. a setting that stays in a deployment slot and is never swapped.  
To do this in Bicep we need to create a `Microsoft.Web/sites/config` resource to set the names of the sticky settings. Then any setting with one of these names will be created as sticky, whether it's added to a slot or to an App Service.  
In our example we need a sticky setting in the staging slot with the name `IsStaging` and the value `true`:
```
resource stickySettings 'Microsoft.Web/sites/config@2022-03-01' = {
  name: 'slotConfigNames'
  parent: app

  properties: {
    appSettingNames: [ 'IsStaging' ]
  }
}

var slotAppSettings = activeApp == 'blue' ? greenAppSettings : blueAppSettings
resource staging 'Microsoft.Web/sites/slots@2022-03-01' = {
  name: 'staging'
  location: location
  parent: app

  properties: {
    siteConfig: {
      linuxFxVersion: linuxFxVersion
      appSettings: concat(slotAppSettings, [ {
            name: 'IsStaging'
            value: true
          } ])
    }
  }
}
```
{: file="infra/modules/appService.bicep" }

> Note that the name of the `Microsoft.Web/sites/config` resource has to be `slotConfigNames`, otherwise you'll get an error.
{: .prompt-warning }


## Time for deployment and swapping !

### First deployment
Now that the Bicep code is ready, to get at the first step of the deployment cycle _for real_, we run the  [`initial-deployment`](https://github.com/xaviermignot/bicep-app-service-slots/blob/main/.github/workflows/initial-deployment.yml) GitHub Actions workflow. This workflow provisions all the resources in Azure, deploys the _blue_ version of the app in the production slot, and the _green_ version in the staging slot.

Basically we now have what is shown on the screenshots earlier. We can also see that the current value of the `activeApp` parameter is visible in the portal: ![Blue activeApp in the Azure portal](/03-deployment-blue.png)_This helps us to implement the rule n°1: save which version is live somewhere_

### Swapping version
Moving forward to the third step of our deployment cycle, we make a first swap by running the [`azcli-swap`](https://github.com/xaviermignot/bicep-app-service-slots/blob/main/.github/workflows/azcli-swap.yml) workflow which uses the following command:  
```sh
az webapp deployment slot swap -g $RG_NAME -n $APP_NAME -s staging
```
{: file=".github/workflows/azcli-swap.yml" }
As expected the `swap` command puts the green version in production:
![The green app in production](/04-green-app-prod.png)
And the blue one is now in staging:
![The blue app in staging](/05-blue-app-staging.png)_As expected the red panel stays in the staging slot_
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
What could happen next ? In case of a rollback, if we run the `azcli-swap` workflow again, the green app will be put back in staging, and the green one in production. That's an handy benefit of using deployment slots: easy to rollback.  
If any change has to be done on the infrastructure _without_ swapping, we just have to get the `activeApp` value of the latest Bicep deployment, and use the same value.  


## Wrapping-up

This marks the end of this post and this series. I hope this gives you a baseline on how to manage your App Services with Bicep and GitHub Actions.  
A always feel free to reach out if you have any question, you can even drop a comment below (this is new). Thanks for reading and happy learning 🤓
