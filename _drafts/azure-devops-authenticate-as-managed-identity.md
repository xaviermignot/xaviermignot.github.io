---
title: Authenticate to Azure DevOps as a managed identity
tags: [azure-devops, azure-pipelines, ci/cd]
img_path: /assets/img/azdo-managed-identity
image:
  path: banner.jpg
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
Using the token is not complicated, basically what's not clear in the official docs is that the access token can be used like a PAT (Personal Access Token).

### With the CLI
When working with the Azure _DevOps_ CLI, there are two options to authenticate:
1. You can pipe the access token to the `az devops login` command, like this in PowerShell: `$token | az devops login` or like this from a Linux shell: `echo $token | az devops login`
2. Or you can set the `AZURE_DEVOPS_EXT_PAT` environment variable and that's it (`$env:AZURE_DEVOPS_EXT_PAT = $token` in PowerShell, `export AZURE_DEVOPS_EXT_PAT="$token"` in a Linux shell.)

And then you can start running [commands](https://learn.microsoft.com/en-us/azure/devops/cli/quick-reference?view=azure-devops).

### With the REST API
This is pretty straightforward as the access token has to be set in the `Authorization` header, for instance in PowerShell (don't forget to replace the `{organization}` placeholder with your organization's name):
```powershell
Invoke-WebRequest -Headers @{ Authorization = "Bearer $token" } -Uri "https://dev.azure.com/{organization}/_apis/projects?api-version=7.2-preview.4"
```
Or from Bash:
```shell
curl -s -H "Authorization: Bearer $token" "https://dev.azure.com/{organization}/_apis/projects?api-version=7.2-preview.4"
```

### As a self-hosted agent
Finally, when configuring a self-hosted agent for Azure Pipelines, you can use the access token as if it was a PAT.  
For a Windows agent, set the `auth` and `token` parameters of the `config.cmd` script:
```powershell
.\config.cmd --auth pat --token $token
```
For a Linux agent, same thing with the `config.sh` script:
```shell
./config.sh --auth pat --token "$token"
```
Refer to the official [docs](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/agents?view=azure-devops&tabs=yaml%2Cbrowser#self-hosted-agents) for the rest of the configuration depending on your scenario.

## Wrapping-up
This concludes this post about managed identity authentication and Azure DevOps. This was a short one as I don't want to repeat what you can find in the official docs, just highlight a few tips that I think can save some time.  
I hope this helps, thanks for reading 🤓
