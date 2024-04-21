---
title: Authenticate to Azure DevOps as a managed identity
tags: [azure-devops, azure-pipelines, ci/cd]
# img_path: /assets/img/azdo-terraform-oidc
# image:
#   path: banner.png
---

Recently I have been fiddling with the Azure DevOps tooling, especially playing with authentication. Yup, I know how to have fun 🤓  
After a [post]({% post_url 2024-04-15-azure-devops-terraform-oidc %}) about workload identity federation and Terraform, let me share some tips about managed identities.  

> If you are not familiar with managed identities, in short it allows to assign identities to Azure resources and let them authenticate without credentials. Go learn about this [here](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview).
{: .prompt-tip }

## Add a managed identity in Azure DevOps
To run the snippets shared in this post, you'll need an Azure resource with a managed identity. For instance, I have a virtual machine with system managed identity.  
You will also need to add the managed identity as a user in your Azure DevOps organization, this can be done by following the steps [here](https://learn.microsoft.com/en-us/azure/devops/integrate/get-started/authentication/service-principal-managed-identity?view=azure-devops).

Once this is ready, you can connect to the resource and start by getting an access token.

## Get an access token
To authenticate against any Azure DevOps tool, we need an access token. How to get one is documented in the official docs, but let me highlight two tips about this.

### Using Azure CLI
The first tip is the magic guid `499b84ac-1321-427f-aa17-267ca6975798`. It's the application id of Azure DevOps's app registration from Microsoft's tenant. Every Entra ID tenant (including yours) has an enterprise application (service principal) linked to this app registration, thus this guid is common to everyone.  

> You can verify this with following command: `az ad sp show --id 499b84ac-1321-427f-aa17-267ca6975798` and see the properties of the service principal
{: .prompt-tip }

Using Azure CLI, we can pass this id to the `--ressource` parameter of the `az account get-access-token` command:
```shell
az login --identity --allow-no-subscriptions
token=$(az account get-access-token --resource 499b84ac-1321-427f-aa17-267ca6975798 --query accessToken -o tsv)
```
By default this command returns an access token to use with Azure, but specifying the `--resource` parameter like this gives a token for Azure _DevOps_.  

> The `--allow-no-subscriptions` flag passed to the `az login` command is necessary if your managed identity doesn't have access to any Azure subscription
{: .prompt-info }

> If you don't want to use a guid, you can use `https://app.vssps.visualstudio.com/` as a resource identifier (this value comes from the `servicePrincipalNames` property of the service principal).
{: .prompt-tip }

### Using authorization endpoint
Second tip, if you don't want to use Azure CLI, there is an authorization endpoint that you can call from any Azure resource linked to a managed identity.  
The key here is to use the `resource` query string parameter with the same guid as with the Azure CLI. You can do it like this in PowerShell:
```powershell
$response = Invoke-WebRequest -Uri 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=499b84ac-1321-427f-aa17-267ca6975798' -Header @{ Metadata = $true }
$token = ($response.Content | ConvertFrom-Json).access_token
```
Or from a Linux shell:
```shell
response=$(curl 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=499b84ac-1321-427f-aa17-267ca6975798' -H Metadata:true -s)
token=$(echo $response | jq -r '.access_token')
```

Now that we have an access token stored in a `$token` variable, let's see how to use it.

## Use the token

### With the CLI

### With the REST API
```shell
authHeader="Authorization: Bearer $token"
curl -s -H "$authHeader" "$ORG_URI""$PROJECT_NAME/_apis/serviceendpoint/endpoints?api-version=7.1-preview.1" | jq
```
### As a self-hosted agent
```powershell
.\config.cmd --auth pat --token $token
```
