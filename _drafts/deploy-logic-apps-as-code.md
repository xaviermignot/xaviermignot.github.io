---
title: "Deploy Azure Logic Apps as code"
tags: [azure, bicep, terraform, logic-apps]
img_path: /assets/img/logic-apps-iac
---

I have been playing with Azure Logic Apps on my own lately, and was wondering how these can be managed by teams in an enterprise environment, especially how can we automate the provisioning and deployment of Logic Apps.  
In this post I will how we can do this using infrastructure as code, with Bicep and Terraform.


## Presentation of the context

As an example we are going to build a very simple project consisting in the following elements:
- A storage account
- A table in the storage account
- A Logic App triggered by HTTP: each request will add a line in the storage table with the IP of the caller

![Diagram](/01-diagram.png) _The following diagram shows what we are going to create here_


## Creating the base resources

To get started we are going to create all the resources but the Logic App itself. If you're not familiar with either Terraform or Bicep this will help to understand the rest of the code, otherwise you can jump to the next section.  
I am using the same *suffix* in all resources names (with a random part to ensure the unicity of the name), and a *prefix* which depends on the resource type.  

> You can use [this page](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations) from the Cloud Adoption Framework as a reference for the abbreviations for Azure resource types.  
This [periodic table of Azure resource types](https://justinoconnor.codes/2022/08/19/azure-periodic-table-of-resource-naming-convention-shorthands/) is also pretty neat.
{: .prompt-tip }  


### Using Bicep
I'm new to Bicep but already used to create a `main.bicep` module containing a *subscription* deployment to create the resource group, and a `resources.bicep` module containing a *resource group* deployment.  
I will focus on the `resources.bicep` module here, here is the beginning of the module with the creation of the storage account, and the storage table:

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

### Using Terraform
For the Terraform version I have put all resources (including the resource group) in a single `main.tf` file.  
Here is the beginning of the file where you can see the random generation for the *suffix*, the creation of the resource group, the storage account and the storage table:

```hcl
resource "random_string" "token" {
  length  = 13
  special = false
  upper   = false
}

locals {
  suffix = "logic-apps-iac-${random_string.token.result}"
}

resource "azurerm_resource_group" "rg" {
  name     = "rg-logic-apps-terraform"
  location = var.location
}

resource "azurerm_storage_account" "storage_account" {
  name                = substr(replace("stor${local.suffix}", "-", ""), 0, 24)
  resource_group_name = azurerm_resource_group.rg.name
  location            = var.location

  account_tier             = "Standard"
  account_replication_type = "LRS"
}

resource "azurerm_storage_table" "storage_table" {
  name                 = "logicAppCalls"
  storage_account_name = azurerm_storage_account.storage_account.name
}
```
{: file="main.tf" }


## Creating the API Connection

To interact with a resource or a service, a logic app requires a *service connection*, which is basically a wrapper around an API, the Azure table storage API in our case.  
This is one of the key step of this post, as the service connection contains the way the logic app authenticates to the storage account. I could have used an account key but decided to use a *managed identity* to follow best practices ([this post](https://medium.com/medialesson/deploying-azure-logic-apps-managed-identity-with-bicep-e1354f185e4d) helped me a lot to make this part work).  

### Using Bicep
Creating the API connection is easy with Bicep, the only trick is to use the `Microsoft.Web/connections@2018-07-01-preview` version to make the managed identity part work:
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


### Using Terraform
Using Terraform is more complex as there is no support for the `Microsoft.Web/connections` resource type in the Azure provider. As a workaround we can use an ARM template like this one:
```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {},
    "variables": {
        "connectionName": "azuretables"
    },
    "resources": [
        {
            "type": "Microsoft.Web/connections",
            "apiVersion": "2018-07-01-preview",
            "name": "tableStorage",
            "location": "[resourceGroup().location]",
            "properties": {
                "parameterValueSet": {
                    "name": "managedIdentityAuth",
                    "values": {}
                },
                "api": {
                    "id": "[concat(subscription().id, '/providers/Microsoft.Web/locations/',resourceGroup().location,'/managedApis/', variables('connectionName'))]"
                }
            }
        }
    ],
    "outputs": {
        "connection": {
            "type": "object",
            "value": {
                "connectionId": "[resourceId('Microsoft.Web/connections', variables('connectionName'))]",
                "connectionName": "[variables('connectionName')]",
                "id": "[concat(subscription().id, '/providers/Microsoft.Web/locations/',resourceGroup().location,'/managedApis/', variables('connectionName'))]"
            }
        }
    }
}
```
{: file="apiConnectionArm.json" }
And invoke the ARM template from the Terraform code using the a `azurerm_resource_group_template_deployment` resource like this:
```hcl
resource "azurerm_resource_group_template_deployment" "api_connection" {
  name                = "deploy-api-connection"
  resource_group_name = azurerm_resource_group.rg.name
  deployment_mode     = "Incremental"
  template_content    = file("apiConnectionArm.json")
}
```
{: file="main.tf" }


## Creating the Logic App workflow

Before jumping into some more IaC code, let's talk about how teams could manage (develop, version and deploy) logic apps and why they are different from other Azure resources like web apps or Azure functions.  
Functions and web apps are code-first services whose deployment require two steps:
- A provisioning step to create the resource in Azure (using IaC, the CLI, the portal, ...)
- A deployment step to push code to the resource (using a CD pipeline, VS Code,  a git push, ...)

These are two distinct steps, and the IaC code used to create the service and the business code pushed to the service are also distinct: deploying a new version of the business code will not make any change to the infrastructure side of the service.  

Logic apps is a design-first service, typically you design your workflow in a GUI (like the Azure portal), and it is saved in a JSON "code" tied to the Azure resource. So there is not the same distinction between provisioning and deployment: any change to the business logic will be reflected on the infrastructure side of the service.  
In other words, the "code" or the "logic" of the logic apps cannot be deployed separately from the creation of the resource itself.  

So how can we develop logic apps from the GUI and automate their deployment using IaC ?  
Here is what I've got in the Azure portal once I've finished developing my logic app:
![Logic App workflow in Azure portal](/02-workflow-portal.png) _A similar experience is available using the VS Code extension_  
Switching to the *Code view* allows me to see my workflow in JSON. From there I can copy the `definition` element and save its content a [file](https://github.com/xaviermignot/deploy-logic-apps-with-iac/blob/81f3560869aaf70bd945132e63b7034833216403/logic_apps/insertIntoTableStorage.json) in my repo.  
Why not taking *all* the content of the code view ? Because the `parameters` element contains the resource id of the API connection (with the subscription id and resource group name) and the storage account name, and I don't want these kind of environment-related information to end up in my git repository.
 
Now that my logic is saved (and versioned) alongside my IaC code, I can use it in the `definition` property of my Logic App workflow:
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

## Wrapping up

