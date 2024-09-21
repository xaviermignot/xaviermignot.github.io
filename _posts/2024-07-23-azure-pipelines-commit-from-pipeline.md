---
title: "Azure DevOps: How to commit from a pipeline"
description: How to make a commit from an Azure pipeline triggered by a CI change, on the same branch.
tags: [azure-devops, azure-pipelines]
media_subpath: /assets/img/azdo-pipeline-commit
image:
  path: banner.webp
  alt: Photo by Rose Galloway Green on Unsplash
date: 2024-07-25 02:00:00
---

This is another post on Azure Pipelines, this time on how to commit from a pipeline. I had to this recently, my use-case was a CI-triggered pipeline who needed to make a commit on the branch it was triggered from.  
Let's see how to do this in a few steps.

## Allow the Build service to make changes on the repo
Before jumping in some yaml, we need to grant the _Contribute_ permission to the Build Service account, aka the identity used by the future pipeline.  
This is done in the project's settings, in the security settings of your repository:
![Repository security settings from Azure DevOps project settings](/project-settings.webp)  
Depending on what you need to do, you might need to set other permissions, like _Create branch_ or _Create tag_. In my case _Contribute_ was enough.

## Allow the pipeline to run Git commands
The first steps of the pipeline should be the following:
```yaml
steps:
  - checkout: self
    persistCredentials: true
  - bash: |
      git config --global user.email "pipeline@dev.azure.com"
      git config --global user.name "Azure DevOps Pipeline"
    name: GitConfig
```
{: file=".azuredevops/azure-pipeline.yml" }
The `checkout` step with `persistCredentials` set to `true` allows the next steps to access to OAuth token for interacting with Azure Repos.  
The `GitConfig` step configures the name and email of the "user" associated to the future commits.

## Make some changes to commit
Then you can add the step(s) that will make the changes to commit. This will depend on your scenario, as an example I will append a line in a log file at the root of the repo:
```yaml
steps:
  # ...Previous steps hidden for readability...
  - bash: |
      echo "$(date -u +"%Y-%m-%dT%H:%M:%SZ") - Pipeline processed by agent $AGENT_NAME" >> pipeline-history.log
    name: ChangeSomethingToCommit
```
{: file=".azuredevops/azure-pipeline.yml" }
Of course my real world scenario was way more interesting than that, and will deserve an upcoming post 😉

## Finally, commit the changes !
This is getting interesting as we add a final step in the pipeline to commit the changes:
```yaml
steps:
  # ...Previous steps hidden for readability...
  - bash: |
      [[ $(git status --porcelain) ]] || { echo "No changes, exiting now..."; exit 0; }
      echo "Changes detected, let's commit them !"
      git add -A
      git commit -m "feat: Update pipeline history from Azure Pipelines [skip ci]"
      branchName=$(echo "$BUILD_SOURCEBRANCH" | sed 's/refs\/heads\///')
      git push origin "HEAD:$branchName"
    name: CommitChanges
```
{: file=".azuredevops/azure-pipeline.yml" }

There are a few interesting things in this script:
- Line 4: the `git status --porcelain` command detects if there is anything to commit, so the script can end early if needed
- Line 7: adding `[skip ci]` in the commit message prevents another pipeline to trigger ([source](https://learn.microsoft.com/en-us/azure/devops/pipelines/repos/azure-repos-git?view=azure-devops&tabs=yaml#skipping-ci-for-individual-pushes)), we want to avoid infinite pipeline loops to trigger 😅
- Line 8: the use of the `Build.SourceBranch` predefined [variable](https://learn.microsoft.com/en-us/azure/devops/pipelines/build/variables?view=azure-devops&tabs=yaml#build-variables-devops-services) with the `sed` command extracts the full name of the branch, even if if contains slashes: `refs/head/feat/your-feature` becomes `feat/your-feature`
- Line 9: when pushing the changes, the pipeline will be in detached head, so the way to overcome this is to use the `git push origin HEAD:name-of-the-branch` syntax

Once again this script fits my case as needed, you might adapt it to fit yours.

## Full pipeline
Here is the full pipeline for the record:
```yaml
trigger:
  branches:
    exclude:
      - main

steps:
  - checkout: self
    persistCredentials: true
  - bash: |
      git config --global user.email "pipeline@dev.azure.com"
      git config --global user.name "Azure DevOps Pipeline"
    name: GitConfig
  - bash: |
      echo "$(date -u +"%Y-%m-%dT%H:%M:%SZ") - Pipeline processed by agent $AGENT_NAME" >> pipeline-history.log
    name: ChangeSomethingToCommit
  - bash: |
      [[ $(git status --porcelain) ]] || { echo "No changes, exiting now..."; exit 0; }
      echo "Changes detected, let's commit them !"
      git add -A
      git commit -m "feat: Update pipeline history from Azure Pipelines [skip ci]"
      branchName=$(echo "$BUILD_SOURCEBRANCH" | sed 's/refs\/heads\///')
      git push origin "HEAD:$branchName"
    name: CommitChanges
```
{: file=".azuredevops/azure-pipeline.yml" }

All the steps were present in the previous snippets, just notice the `trigger` who prevents the pipeline to run on the `main` branch, as committing on the main requires a pull request. In my real world scenaro there is also a path filter so that the pipeline is triggered only when selected files are changed. And yes, this is also another thing that depends on your use-case.

To show that it works, triggering the pipeline will add commits like this one in the history:
![Commit details from Azure Repos](/commit-details.webp)_Notice the Azure DevOps Pipeline "user"_

## Wrapping-up
In this post we have seen how to make a commit from a pipeline triggered by a CI change. Nothing new or fancy, but a few gotchas along the way, and something that can be useful in many situations. Thanks for reading 🤓