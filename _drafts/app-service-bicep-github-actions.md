---
title: "Deploy App Services with Bicep & GitHub Actions"
tags: [azure, bicep, app-service, github-actions]
img_path: /assets/img/app-service-bicep-github-actions
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
{: file="infra/modules/appServicePlan.bicep" }

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

output appServiceName string = app.name // The name of the App Service is returned for later use in the workflows
```
{: file="infra/modules/appService.bicep" }

The Bicep code is now ready so we will see how to run it using GitHub Actions.


## The GitHub Actions workflows

In this section we are going to build a single workflow in GitHub Actions to provision our App Service and deploy code to it.  
First, we must establish a link between our Azure environment and our GitHub account.

### Grant GitHub Actions access to our Azure subscription

We are going to use the Azure Login action in our workflow with a service principal. The set-up is explained in the docs, follow the instructions [here](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-cli%2Clinux#use-the-azure-login-action-with-a-service-principal-secret) to create the service principal and set the `AZURE_CREDENTIALS` secret.  
While we are at setting secrets, add a secret `AZURE_SUBSCRIPTION` with your subscription's id and `AZURE_REGION` with the Azure region you want to deploy your resources to.

### A first job for provisioning the resources

Let's create a GH workflow to run our Bicep code: the first notable step authenticates to Azure using the `azure/login` action and the secrets we have just created.  
The next steps will use this authentication so we can add another step for running the Bicep template with the `azure/arm-deploy` action. We specify the scope, template file, parameters and the workflow is ready:
{% raw %}
```yaml
name: Provision Azure resources
on:
  workflow_dispatch:
jobs:
  provision:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      
      - name: Bicep deploy
        uses: azure/arm-deploy@v1
        with:
          scope: subscription
          subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION }}
          region: ${{ secrets.AZURE_REGION }}
          template: ./infra/main.bicep
          parameters: project=bicep-aps-demo location=${{ secrets.AZURE_REGION }}
          deploymentName: apsBicepDemo
```
{: file=".github/workflows/simple-workflow.yml" }
{% endraw %}
The `workflow_dispatch` trigger allows us to manually run the workflow from the GitHub UI, once it's done the resources will be created and the website look like this:
![Empty App Service](01-empty-app.png)  
For now it's an empty shell waiting for our content, we can notice the _Build with .NET_ line, it confirms that the application stack has been properly set, as we can also see in the Azure portal:
![Application Stack in Azure portal](02-portal-application-stack.png)

### Time for code deployment !
To push a package to our new web app we are going to add a second job to our workflow with the following steps/actions:
- `dotnet publish` to restore, build and package the code in a single command
- `azure/login` like in the provision job to authenticate to Azure
- `azure/webapps-deploy` to push the package to the App Service

Let's focus on the latest step, the most important but yet simple:
{% raw %}
```yaml
- name: Deploy to App Service
  uses: azure/webapps-deploy@v2
  with: 
    app-name: ${{ needs.provision.outputs.appServiceName }} 
    package: ./src/ApsSlotsDemo.App/publish/ApsSlotsDemo.App
```
{: file=".github/workflows/simple-workflow.yml" }
{% endraw %}
A few words about authentication: there are several ways to authenticate for App Service deployment (check the official [documentation](https://learn.microsoft.com/en-us/azure/app-service/deploy-github-actions?tabs=applevel)). I have chosen to use a _service principal_, basically the same identity is used for provisioning the resources and deploying the code.  
That's why there is no credential parameter, authentication is handled at the previous step (`azure/login`).  

Notice also that we need to pass the name of the App Service to deploy to. That's because the service principal authentication doesn't target a specific App Service, unlike publish profiles.  
As the name of the App Service is generated during the provisioning job, we need to pass it from one job to another. To do this we have to set an `id` to the `Bicep deploy` add an output to the `provision` job like this:
{% raw %}
```yaml
    outputs:
      appServiceName: ${{ steps.bicep-deploy.outputs.appServiceName }}
```
{% endraw %}
Then in the `deploy` job, we add the `needs: provision` line at the job level to indicate that the `provision` must complete first, and to allow the use of its outputs.

Overall the complete `deploy` job looks like this:
{% raw %}
```yaml
  deploy:
    runs-on: ubuntu-latest
    needs: provision
    defaults:
      run:
        working-directory: src/ApsSlotsDemo.App
    steps:
      - uses: actions/checkout@v3
        with:
          ref: blue

      - name: dotnet publish
        run: dotnet publish -c Release -o publish/ApsSlotsDemo.App

      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Deploy to App Service
        uses: azure/webapps-deploy@v2
        with: 
          app-name: ${{ needs.provision.outputs.appServiceName }} 
          package: ./src/ApsSlotsDemo.App/publish/ApsSlotsDemo.App
```
{: file=".github/workflows/simple-workflow.yml" }
{% endraw %}

The whole workflow can be found in the GitHub [repo](https://github.com/xaviermignot/bicep-app-service-slots/blob/main/.github/workflows/simple-workflow.yml).

Finally if you run this workflow and browse the app again you should see the deployed content:
![Deployed app](03-deployed-app.png) _Wondering what this blue app thing is about ? This is the topic of the next post_

## Wrapping-up

