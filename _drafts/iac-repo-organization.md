---
title: "How to organize a repo with IaC, application code and CI/CD"
tags: [azure, bicep, terraform, github-actions]
---

## Organization of the repository

To organize my repo who contains both IaC and application code, I like to proceed like this:
- An `infra` folder contains the IaC
- A `src` folder contains the application code
- Another folder contains the CI/CD pipelines, it can be:
  - `.github/workflow` for GitHub Actions (this is what we will be using here)
  - `.azuredevops` or `.azdo` for Azure DevOps

It leaves rooms at the root for the README file, license and changelog. I can also add a `tests` folder at the root or in the `src` folder for the application code tests.


## The Bicep code

For the Bicep code I like to apply what I've learned from the Azure Developer CLI, so the code in the `infra` folder is organized like this:
- The `main.bicep` is the starting point, it's our main deployment whose scope is our Azure subscription.
- The `resources.bicep` module called by the `main` is scoped to a resource group created by the `main`.
- The `infra/modules` folder contains various modules called by the `resources` module. The organization of the modules varies from project to project.

