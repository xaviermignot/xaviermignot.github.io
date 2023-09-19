---
title: "Azure Pipelines: Storing objects in variables from a PowerShell task"
tags: [azure-devops, azure-pipelines, ci/cd]
img_path: /assets/img/azdo-pipeline-object
image:
  path: banner.jpg
  alt: Photo by karosu on Unsplash
---

In this quick post I will share a trick I have found recently to store an object in an Azure Pipelines variable from a PowerShell script.  

## The problem
Setting a variable in a script for later use in an Azure DevOps' pipeline is possible using the `task.setvariable` command as described here in the [docs](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/set-variables-scripts?view=azure-devops&tabs=powershell).  
This works great for simple variables like this:
```yaml
steps:
  - pwsh: |
      Write-Host "##vso[task.setvariable variable=variableName]variableValue"
  - pwsh: |
      Write-Host "Value from previous step: $(variableName)"
```
But it's a little bit trickier for complex variables likes objects, arrays, or object arrays.  
As an example, let's say I want to retrieve the name, type and resource group of all the resources in an Azure subscription like this:
```powershell
$resources = Get-AzResource | Select-Object -Property Name,Type,ResourceGroupName
```
Using this line and replacing `variableValue` with `$resources` in the previous yaml snippet will not work as you can check from your terminal:
![View from my terminal](/01-terminal.png)_I have cropped the image for obvious sensitive reasons_
You see that a complex variable in a string is empty, even if tabular data is shown when the variable is typed alone in a terminal line. As you can guess the same code in a pipeline will result in an empty variable, and fortunately there is a simple solution to this.

## The solution
To store the list of resources in variable, we simply serialize it in JSON and put it on a single like this:
```powershell
$resourcesJson = $resources | ConvertTo-Json -Compress
```
Notice the `-Compress` flag which puts all the JSON on a single line.  
In a later step the JSON is deserialized like that:
```powershell
$resources = '$(resources)' | ConvertFrom-Json
```
This is how it looks in a simple yet complete pipeline:
```yaml
pool:
  vmImage: ubuntu-latest

steps:
  - task: AzurePowerShell@5
    inputs:
      azureSubscription: $(azureServiceConnection)
      azurePowerShellVersion: LatestVersion
      ScriptType: InlineScript
      Inline: |
        $resources = Get-AzResource | Select-Object -Property Name,Type,ResourceGroupName
        $resourcesJson = $resources | ConvertTo-Json -Compress
        Write-Host "##vso[task.setvariable variable=resources]$resourcesJson"
  - pwsh: |
      $resources = '$(resources)' | ConvertFrom-Json
      Write-Host "There are $($resources.Count) resources in the list"
```
And the working result from the Azure Pipelines UI:
![Azure DevOps](/02-azdo.png)

## Wrapping up
As conclusion, let me explain the real-world scenario where I needed this trick, as you might wonder why we do this instead of putting everything in a single script.  

Well, I needed to perform some operations on a set of virtual machines, with waiting times between them. To avoid using the Azure Pipelines agents in the waiting times, I have used [agentless jobs](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/phases?view=azure-devops&tabs=yaml#agentless-tasks). Thus I have split my pipeline in several jobs, the actual jobs running on my agents, and the [Delay](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/delay-v1?view=azure-pipelines) tasks handled by agentless jobs.  
The role of the first job in my pipeline was to retrieve the list of targeted machines, and store in a variable to ensure the same set of machines was used for the whole pipeline.  

That's why I have simplified the examples in this post with a single-job pipeline. I can't recommend enough to carefully read the `task.setvariable` [documentation](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/set-variables-scripts?view=azure-devops&tabs=powershell), as there are some subtle differences whether you're setting a variable within a job, for another job or for another stage.