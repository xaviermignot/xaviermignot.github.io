---
title: "Deploy Azure Logic Apps as code"
tags: [azure, bicep, terraform, logic-apps]
img_path: /assets/img/logic-apps-iac
---

I have been playing with Azure Logic Apps on my own lately, and was wondering how these can be managed by teams in an enterprise environment, especially how can we automate the provisioning and deployment of Logic Apps.  
In this post I will show how we can do this using infrastructure as code using Bicep.


## Presentation of the context

As an example we are going to build a very simple project consisting in the following elements:
- A Storage Account
- A table in the Storage Account
- A Logic App triggered by HTTP: each request will add a line in the storage table with the IP of the caller

![Diagram](/01-diagram.png) _The following diagram shows what we are going to create here_


## GitHub repository

I have prepared a [GitHub repository](https://github.com/xaviermignot/deploy-logic-apps-with-iac/) to let you see the whole code of the demo, and eventually run it by yourself.  
The repo contains the full IaC code in Bicep and Terraform, in the post I will show the most relevant parts in Bicep only for clarity reasons (and also because it's all little bit more complicated in Terraform, more on this later).

## Creating the base resources

To get started we are going to create all the resources but the Logic App itself. If you're not familiar with Bicep this will help to understand the rest of the code, otherwise you can jump to the next section.  
Note that my naming convention is composed of the same *suffix* in all resources names (with a random part to ensure the unicity of the name), and a *prefix* depending on the resource type.  

> You can use [this page](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations) from the Cloud Adoption Framework as a reference for the abbreviations for Azure resource types.  
This [periodic table of Azure resource types](https://justinoconnor.codes/2022/08/19/azure-periodic-table-of-resource-naming-convention-shorthands/) is also pretty neat.
{: .prompt-tip }  

I'm new to Bicep but already used to create a `main.bicep` module containing a *subscription* deployment to create the resource group, and a `resources.bicep` module containing a *resource group* deployment.  
I will focus on the `resources.bicep` module here, here is the beginning of the module with the creation of the Storage Account, and the storage table:

```
@description('The suffix to use in resource naming.')
param suffix string
param location string

resource storageAccount 'Microsoft.Storage/storageAccounts@2021-09-01' = {
  name: substring('stor${replace(suffix, '-', '')}', 0, 24)
  location: location

  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
}

resource tableStorageService 'Microsoft.Storage/storageAccounts/tableServices@2021-09-01' = {
  name: 'default'
  parent: storageAccount
}

resource storageTable 'Microsoft.Storage/storageAccounts/tableServices/tables@2021-09-01' = {
  name: 'logicAppCalls'
  parent: tableStorageService
}
```
{: file="resources.bicep" }


## Creating the API Connection

To interact with a resource or a service, a Logic App requires a *service connection*, which is basically a wrapper around an API, the Azure table storage API in our case.  
This is one of the key steps of this post, as the service connection contains the way the Logic App authenticates to the Storage Account. I could have used an account key but decided to use a *managed identity* to follow best practices ([this post](https://medium.com/medialesson/deploying-azure-logic-apps-managed-identity-with-bicep-e1354f185e4d) helped me a lot to make this part work).  

Long story short, the trick with the Managed Identity is to use the `2018-07-01-preview` API version of the `Microsoft.Web/connections` resource type. Bicep will show a warning because of this but it's currently the only way to make it work:
```
resource tableStorageConnection 'Microsoft.Web/connections@2018-07-01-preview' = {
  name: 'tableStorage'
  location: location

  properties: {
    parameterValueSet: {
      name: 'managedIdentityAuth'
      values: {}
    }
    api: {
      id: '${subscription().id}/providers/Microsoft.Web/locations/${location}/managedApis/azuretables'
    }
  }
}
```
{: file="resources.bicep" }


## Creating the Logic App workflow

Before jumping into some more IaC code, let's talk about how teams could manage (develop, version and deploy) Logic Apps and why they are different from other Azure resources like Web Apps or Azure Functions.  
Functions and Web Apps are code-first services whose deployment require two steps:
- A provisioning step to create the resource in Azure (using IaC, the CLI, the portal, ...)
- A deployment step to push code to the resource (using a CD pipeline, VS Code,  a git push, ...)

These are two distinct steps, and the IaC code used to create the service and the business code pushed to the service are also distinct: deploying a new version of the business code will not make any change to the infrastructure side of the service.  

Logic apps is a design-first service, typically you design your workflow in a GUI (like the Azure portal), and it is saved in a JSON "code" tied to the Azure resource. So there is not the same distinction between provisioning and deployment: any change to the business logic will be reflected on the infrastructure side of the service.  
In other words, the "code" or the "logic" of the Logic Apps cannot be deployed separately from the creation of the resource itself.  

So how can we develop Logic Apps from the GUI and automate their deployment using IaC ?  
Here is what I've got in the Azure portal once I've finished developing my Logic App:
![Logic App workflow in Azure portal](/02-workflow-portal.png) _A similar experience is available using the VS Code extension_  
Switching to the *Code view* allows me to see my workflow in JSON. From there I can copy the `definition` element and save its content a [file](https://github.com/xaviermignot/deploy-logic-apps-with-iac/blob/81f3560869aaf70bd945132e63b7034833216403/logic_apps/insertIntoTableStorage.json) in my repo.  
Why not taking *all* the content of the code view ? Because the `parameters` element contains the resource id of the API connection (with the subscription id and resource group name) and the Storage Account name, and I don't want these kind of environment-related information to end up in my git repository.
 
Now that my logic is saved (and versioned) alongside my IaC code, I can use it in the `definition` property of my Logic App workflow (using the `loadJsonContent` builtin function):
```
resource logicApp 'Microsoft.Logic/workflows@2019-05-01' = {
  name: 'ala-${suffix}'
  location: location

  identity: {
    type: 'SystemAssigned'
  }

  properties: {
    definition: loadJsonContent('../logic_apps/insertIntoTableStorage.json')
    parameters: {
      '$connections': {
        value: {
          '${connectionApiName}': {
            connectionId: tableStorageConnection.id
            connectionName: connectionApiName
            id: '${subscription().id}/providers/Microsoft.Web/locations/${location}/managedApis/${connectionApiName}'
            connectionProperties: {
              authentication: {
                type: 'ManagedServiceIdentity'
              }
            }
          }
        }
      }
      storageAccountName: {
        value: storageAccount.name
      }
    }
  }
}
```
{: file="resources.bicep" }

The only thing left to do is to grant the Logic App access to the Storage Account using an assignment to the `Storage Table Data Contributor` builtin role. This can be seen [here](https://github.com/xaviermignot/deploy-logic-apps-with-iac/blob/81f3560869aaf70bd945132e63b7034833216403/bicep/resources.bicep#L73-L83) in the repository.  

Finally everything can be deployed using a simple `az deployment sub create` command.


## What about Terraform ?

As mentioned earlier, this little demo project has been done in Bicep and Terraform, but I ended up keeping only the Bicep version in the blog post. This is for clarity reasons but also because doing this with Terraform has the following drawbacks:
1. There is no support for API connections in the AzureRM Terraform provider, so I had to create [an ARM template](https://github.com/xaviermignot/deploy-logic-apps-with-iac/blob/81f3560869aaf70bd945132e63b7034833216403/terraform/apiConnectionArm.json) and [invoke it](https://github.com/xaviermignot/deploy-logic-apps-with-iac/blob/81f3560869aaf70bd945132e63b7034833216403/terraform/main.tf#L30-L35) in my Terraform code to create the API connection
2. A Logic App workflow can be created using the `logic_app_workflow` resource of the AzureRM provider, but it doesn't provide a way to set the definition of the workflow. The solution was to use this resource to create the workflow (with the Managed Identity for later role assignment), and then another ARM template [file](https://github.com/xaviermignot/deploy-logic-apps-with-iac/blob/81f3560869aaf70bd945132e63b7034833216403/terraform/logicAppWorkflowArm.json) to deploy the definition of the Logic App from the Terraform [code](https://github.com/xaviermignot/deploy-logic-apps-with-iac/blob/81f3560869aaf70bd945132e63b7034833216403/terraform/main.tf#L59-L74).


## Wrapping up

