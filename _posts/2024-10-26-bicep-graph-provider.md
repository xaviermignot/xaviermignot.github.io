---
title: First look on the Graph provider for Bicep
description: Provision an Azure App Service protected by Entra ID using Bicep only.
tags: [azure, bicep, app-service]
media_subpath: /assets/img/bicep-graph-provider
image:
  path: banner.webp
  alt: Image generated using Microsoft Designer
date: 2024-10-26 10:00:00
---

I have made a [post]({% post_url 2024-02-05-terraform-vs-bicep %}) a few month ago to compare Terraform/OpenTofu and Bicep in a Azure context. I was mentioning in this post that interacting with Azure AD/Entra ID was not possible with Bicep, but also put a link to a GitHub issue in the conclusion about an upcoming _provider_ for Microsoft Graph.  
Time has passed since then and the provider has been released in public preview for testing. Let's see how it works using a very common use-case.

## An App Service protected by Easy Auth using Bicep ONLY
Easy Auth is the built-in authentication feature of Azure App Service. In a nutshell it forces the users to authenticate with an Identity Provider before redirecting them to the app. All of this without the need to modify the code of the app.  
Using Entra ID as the Identity Provider makes a good use case for Bicep Graph provider as we will have to create Azure resources (App Service and App Service Plan) using _classic_ Bicep and Microsoft Graph resources (App Registration and Enterprise App) using the Graph provider.

![The resources provisioned by the sample](resources.webp){: width="500" }_This diagram shows the resources to provision for this sample_

> If you are not familiar with App Service Easy Auth, you can start by reading [this page](https://learn.microsoft.com/en-us/azure/app-service/overview-authentication-authorization) in the docs
{: .prompt-tip }

## GitHub repository
The full code associated with this post is available on my GitHub [here](https://github.com/xaviermignot/bicep-app-service-easy-auth). You'll find instructions there on how to run the code in your Azure environment.  

## How the Microsoft Graph provider for Bicep works
As the Graph provider is in preview, theses lines need to be added in the `bicepconfig.json` file:
```json
{
  "experimentalFeaturesEnabled": {
      "extensibility": true
  }
}
```
{: file="bicepconfig.json" }
This enables the _extensibility_ feature of Bicep, allowing other _providers_ than Azure Resource Manager to be used. The Microsoft Graph provider is just one provider, other providers are on the way.  

The way the provider is invoked is interesting, as the Bicep execution mode remains the same: a single API call is made from the client machine to create an ARM deployment, and the rest happens in Azure, even the execution of the Graph provider.  

On the Azure side, Azure Resource Manager uses its _deployment engine_ to provision Azure resources (by default) and MS Graph resources (when the specified provider is `microsoftGraph`). So the calls to the MS Graph API are not made from the client machine, but from the deployment engine running in Azure.  

I have tried to illustrate the concept with the following diagram:
![The workflow of the Bicep Graph extension](/workflow.webp)_This diagram is heavily inspired by the one on [this page](https://learn.microsoft.com/en-us/graph/templates/overview-bicep-templates-for-graph)_

> We can compare the Bicep providers to those used by Terraform/OpenTofu: both allow the use of external APIs but in the Terraform/OpenTofu world, the providers run on the client machine
{: .prompt-info }

## Show me some Bicep Graph code !
To specify the MS Graph provider, the first line of a Bicep file must be `extension microsoftGraph`. Otherwise the default provider is used, which is obviously Azure Resource Manager.  
For instance this module creates the App Registration required for Easy Auth:
```
extension microsoftGraph // This is important, it says "use the MS Graph provider for this file"

param project string
param defaultHostName string // This comes from the App Service module (who uses the ARM default provider)

resource app 'Microsoft.Graph/applications@v1.0' = {
  displayName: 'app-${project}'
  uniqueName: 'app-${project}'

  api: {
    requestedAccessTokenVersion: 2
  }

  web: {
    redirectUris: [
      // Sets a redirect URI to the default hostname (*.azurewebsites.net)
      uri('https://${defaultHostName}', '.auth/login/aad/callback')
    ]

    implicitGrantSettings: {
      enableAccessTokenIssuance: true
      enableIdTokenIssuance: true
    }
  }

  requiredResourceAccess: [
    {
      resourceAppId: '00000003-0000-0000-c000-000000000000' // Microsoft Graph
      resourceAccess: [
        {
          id: '37f7f235-527c-4136-accd-4a02d197296e' // User sign-in
          type: 'Scope'
        }
      ]
    }
  ]
}

output clientId string = app.appId
```
{: file="modules/appRegistration.bicep" }
I have written Terraform code to provision App Service with Easy Auth a while ago, and the App Registration [code](https://github.com/xaviermignot/tf-general-poc/blob/main/eng/tf/appgw-app-service-easy-auth/az-ad.tf) in HCL is very similar to the above code.  

What is really cool is that it's just a resource defined in Bicep, so it integrates seamlessly with the other (ARM) resources in the code. For instance the `defaultHostname` parameter comes from the App Service resource in my Bicep code, and the `clientId` output from above is passed to the module in charge of the Easy Auth ARM resource:
```
param appServiceName string // This comes from the App Service module (who uses the ARM default provider)
param clientId string // This comes from the App Registration modules (who uses the MS Graph provider)

resource app 'Microsoft.Web/sites@2023-12-01' existing = {
  name: appServiceName
}

resource easyAuth 'Microsoft.Web/sites/config@2023-12-01' = {
  parent: app
  name: 'authsettingsV2'
  properties: {
    globalValidation: {
      requireAuthentication: true
      redirectToProvider: 'azureActiveDirectory'
      unauthenticatedClientAction: 'RedirectToLoginPage'
    }
    identityProviders: {
      azureActiveDirectory: {
        enabled: true
        registration: {
          clientId: clientId
          openIdIssuer: uri(environment().authentication.loginEndpoint, tenant().tenantId)
          clientSecretSettingName: 'MICROSOFT_PROVIDER_AUTHENTICATION_SECRET'
        }
        validation: {
          defaultAuthorizationPolicy: {
            allowedApplications: [
              clientId
            ]
          }
        }
      }
    }
  }
}
```
{: file="modules/easyAuth.bicep" }

## A look from the Azure portal
Leaving the IDE for the Azure portal, at the resource group level the deployment for the App Registration is just another deployment among the others:
![The list of deployments at the resource group level in Azure portal](portal-rg.webp)_You know, I'm something of an ARM template myself_

Zooming on this deployment, the _Deployment details_ from the _Overview_ blade is buggy for now and doesn't show the list of resources but in the _Template_ blade we can see that the deployed resource is of type `Microsoft.Graph/applications`:
![The ARM template for the App Registration](portal-template.webp)_Always nice to see some ARM from time to time, makes me love Bicep even more 💖_

Once again it is kinda fun to see how this extensibility feature has been implemented by the Bicep team, allowing interactions with other APIs than ARM while keeping the core workflow of Bicep the same, that's pretty clever 🤓

## Wrapping-up
In this post we have seen how we can use the Graph provider for Bicep, and a glimpse on how Bicep extensibility feature works under the hood. Personally I think it's great to see how Bicep is evolving in its own way, amazing work done by the Bicep team, I can't wait to see what's coming next 🤓
