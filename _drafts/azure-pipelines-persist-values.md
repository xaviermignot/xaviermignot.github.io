---
title: How to persist values between Azure Pipelines runs
tags: [azure-devops, azure-pipelines]
media_subpath: /assets/img/azdo-persist-values
---

Using Azure Pipelines, it's pretty common to store values in variables for later use. Whether the value is saved for the next step, job or stage, it can be done easily using the `task.setvariable` command as documented [here](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/set-variables-scripts) (I have also share a tip about this in [this post]({% post_url 2023-09-18-azure-pipelines-objects-variables %})).  
In this post we will see how we can store values in variable groups to persist them between pipeline runs.

## What are variable groups ?
Variable groups are located in the _Library_ section of the Azure Pipelines' UI:
![](/variable-groups-ui.png)
Each variable group can contain several variables, and can be used by one or many pipelines within the same project. I use them to store environment-related values I don't want to put in my repos, such as Azure subscription ids for example.  

> Variables in groups can be marked as sensitive to mask their value in the pipelines logs. A group can also be bound to an Azure key vault so that its variables are retrieved from the vault's secrets.
{: .prompt-info }

## A simple use case sample
As an example, let's say we need to store the outputs of an Azure deployment in a variable group. The [AzureResourceManagerTemplateDeployment](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/azure-resource-manager-template-deployment-v3) task has a `deploymentOutputs` input for that: we can provide a variable name here, and it will contain the outputs of the deployment in JSON:
```yaml
steps:
  - task: AzureResourceManagerTemplateDeployment@3
    inputs:
      # ..lines removed for clarity, this input is the most important here...
      deploymentOutputs: outputs
```
{: file=".azuredevops/azure-pipeline.yml" }

## Setting the variables in the group
To set the variables I'm going to use the Azure DevOps CLI. It's a simple task but there is a few things to mention:
- There is no `set` command in the `az pipelines variable-group variable` command group, but a `create` and and `update` one. We need to use the right command depending on whether the variable already exists
- Listing the variables in a group is possible using the `az pipelines variable-group variable list` command, but it requires the id of group. The same result can be achieved from the group name using the `az pipelines variable-group list` command with a JMESPATH query.

Here is the full PowerShell script I use for this:
```powershell
param(
  [string]$Outputs,
  [string]$VariableGroupName,
  [string]$AzDoOrgUrl,
  [string]$AzDoProjectName
)

# Sets the default org and project for all az devops cli commands
az devops configure --defaults organization=$AzDoOrgUrl project=$AzDoProjectName

# Gets the variable group id and all variables from the group name
$variableGroup = $(az pipelines variable-group list --query "[?name=='$VariableGroupName'].{id:id,variables:variables}") | ConvertFrom-Json
# Extracts the name of the existing variables
$existingVariableNames = Get-Member -InputObject $variableGroup.variables -MemberType NoteProperty | Select-Object -ExpandProperty Name

$parsedOutputs = $Outputs | ConvertFrom-Json
# For each output, create or update the variable in the group
Get-Member -InputObject $parsedOutputs -MemberType NoteProperty | Select-Object -ExpandProperty Name | ForEach-Object {
  if ($existingVariableNames -contains $_) {
    az pipelines variable-group variable update --group-id $variableGroup.id --name $_ --value $parsedOutputs.$_.value
  }
  else {
    az pipelines variable-group variable create --group-id $variableGroup.id --name $_ --value $parsedOutputs.$_.value
  }
}
```
{: file=".azuredevops/Set-OutputsInVariableGroup.ps1" }

Note that authentication is required to run the script. I have deliberately omitted from the script, so that you can test it locally after authenticating using a PAT.  
For the pipeline, authentication will be handled in the YAML, let's see how to do that.

## Running the script from the pipeline
Again, running this script should be easy but there is a little twist because of the authentication. 

### Using the Build Service account
The easiest way is to use the `$(System.AccessToken)` predefined variable and pipe it to the `az devops login` command:
```yaml
steps:
  # ...previous task removed for clarity...
  - pwsh: |
      $env:ACCESS_TOKEN | az devops login 
      .azuredevops/Set-OutputsInVariableGroup.ps1 `
        -Outputs $env:OUTPUTS `
        -VariableGroupName '<NAME OF THE GROUP HERE>' `
        -AzDoOrgUrl $env:ORG_URL `
        -AzDoProjectName $env:PROJECT_NAME
    env:
      OUTPUTS: $(ouputs)
      ORG_URL: $(System.CollectionUri)
      PROJECT_NAME: $(System.TeamProject)
      ACCESS_TOKEN: $(System.AccessToken)
```
{: file=".azuredevops/azure-pipeline.yml" }

> Notice the use of the `$(System.CollectionUri)` and `$(System.TeamProject)` predefined variables to get the organization URL and the project name
{: .prompt-info }

Using the `$(System.AccessToken)` variable will make the _Build Service_ account of your project authenticate against Azure DevOps. By default this service account has the _Reader_ role on your variable group, so before running the pipeline you need to change this to _Administrator_:
![](/variable-group-security.png)

After that the pipeline should run without any issue, and the variable group will be updated by the service account with all the deployment outputs as variables.

### Using another account
If you can't use the Build Service account (because your organization forbids it), it's still possible using a service principal (for instance, the one you use for deploying resources in Azure).  
You will need to:
- Add the service principal as a user in the Azure DevOps organization
- Give it the _Administrator_ role to the variable group (just like for the Build Service account)

> The service principal will need the _Basic_ access level to the Azure DevOps organization so it will use a licence (there are 5 free users for each org)
{: .prompt-warning }

In the pipeline, a simple PowerShell task is not enough, an `AzureCLI` is required, and a token has be retrieved in order to authenticate against the DevOps CLI using the following lines:
```powershell
$token = $(az account get-access-token --resource 499b84ac-1321-427f-aa17-267ca6975798 --query accessToken -o tsv)
$token | az devops login
```

The full task in the pipeline will look like this:
```yaml
steps:
  # ...previous task removed for clarity...
  - task: AzureCLI@2
    inputs:
      azureSubscription: <NAME OF SERVICE CONNECTION HERE>
      scriptType: pscore
      scriptLocation: inlineScript
      inlineScript: |
        $token = $(az account get-access-token --resource 499b84ac-1321-427f-aa17-267ca6975798 --query accessToken -o tsv)
        $token | az devops login
        .azuredevops/Set-OutputsInVariableGroup.ps1 `
          -Outputs $env:OUTPUTS `
          -VariableGroupName '<NAME OF THE GROUP HERE>' `
          -AzDoOrgUrl $env:ORG_URL `
          -AzDoProjectName $env:PROJECT_NAME
    env:
      OUTPUTS: $(outputs)
      ORG_URL: $(System.CollectionUri)
      PROJECT_NAME: $(System.TeamProject)
```
{: file=".azuredevops/azure-pipeline.yml" }

> The code snippet above will also work if you are using a managed identity instead of a service principal
{: .prompt-tip }

## Wrapping up