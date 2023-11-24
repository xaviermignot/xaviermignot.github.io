---
title: "Host your org's GitHub runners in Azure Container Apps"
tags: [github-actions, azure, container-apps, bicep]
img_path: /assets/img/gh-runner-aca
---

Getting started with GitHub Actions is easy with the GitHub-hosted runners. But once the testing phase done, most organizations want to use their own machines to run the workflows for customization, governance, security and/or cost reasons.  
What if your could use a managed service like Azure Container Apps instead of virtual machines for that ? In this post I will show you how to do that with Bicep and GitHub Actions.  
This is a two-part series, the second part focuses on the scaling aspect of Container Apps.

> Note that this post is dedicated to runners for an _organization_, not for your _personal_ account. There are still many things in common, and some differences (regarding authentication mostly).
{: .prompt-info }

> If you need an introduction to Azure Container Apps, the best place is the official [documentation](https://learn.microsoft.com/en-us/azure/container-apps). Also the GitHub docs about self-hosted runners is [here](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners) if you need.
{: .prompt-tip }

## GitHub repository and organization
For the purpose of this series I have created my very own GitHub organization (for free, you can [create](https://github.com/account/organizations/new) your own too !), and carefully crafted [a repository](https://github.com/xmi-cs/aca-gh-actions-runner). I will show only the most important snippets of the code in the post, you can still browse the repo for more context.  
The repo contains a Dockerfile, GitHub workflows and Bicep code to create the necessary resources in Azure, and run a "hello world" Azure CLI script on the self-hosted runners. 

![Architecture diagram](/01-diagram.png) _This is what the architecture looks like_

> If you need a step-by-step, tutorial approach, you can follow the instructions in the [README](https://github.com/xmi-cs/aca-gh-actions-runner/blob/main/README.md) of the repo. In the rest of the post I will explain _how it works_, not _how to deploy it_ in your organization.  
{: .prompt-info }

## Create a Dockerfile (or use another one as a base)
The first thing that surprised me when I started working on this, is that there is no official Docker image provided by GitHub.  
You can either create you own Dockerfile from an OS base (as I first did [here](https://github.com/xmi-cs/aca-gh-actions-runner/blob/main/src/Dockerfile.from-ubuntu)), or use a third-party as a base like one from this excellent [repo](https://github.com/myoung34/docker-github-actions-runner). I choose the second option so my Dockerfile is very simple:
```dockerfile
FROM myoung34/github-runner:ubuntu-jammy

# install Azure CLI
RUN curl -sL https://aka.ms/InstallAzureCLIDeb | bash
```
The base container image is very flexible and requires several environment variables, but we'll get into that later.

## Create a GitHub App
Self-hosted runners use the GitHub REST API to register themselves and query to jobs they have to run. For organizations, the recommended authentication method is to use a GitHub App: the workflows will use a private key to generate tokens to use to authenticate against the REST API.  

The creation a GitHub App starts from the _Developer Settings_ of your organization. The most important settings are the _permissions_, for this project we need the following scopes:
- In _Repository permissions_:
  - Set `Actions` to `Read-only`
  - Set `Metadata` to `Read-only` (it should be selected by default)
  - Set `Variables` to `Read and write`. This is not required for the runners but for one of my workflows
- In _Organization permissions_:
  - Set `Administration` to `Read-only`
  - Set `Self-hosted runners` to `Read and write`

After that the process prompts us to generate a private key that the browser downloads as a `.pem` file, and to install the app in our organization (for all or a selection of repositories).

> The private key is used to generate access tokens with the identity of the GitHub App. Ideally it should go in a Key Vault but for this demo I have chose to put it in a Action secret.
{: .prompt-warning }

## Deploy the Azure resources
It's time to provision some stuff in Azure, starting with:
- A container registry to store your container images
- A Container Apps [environment](https://learn.microsoft.com/en-us/azure/container-apps/environment), to define the pricing, networking and logging settings of one or several Container Apps (one in our case)
- A Log Analytics Workspace to send your logs to. This is optional but very handy in case of troubleshooting

You can create these from the portal, or if you have forked the repo run the workflow that does this for you.
Then after building and pushing the container image to the registry (using the `az acr build` command) we can move on to create the Container App.

## Provision the runners
For the creation of the Container App I will walk through the GitHub Action workflow that as there are a few _gotchas_ along the way.

### Generate an access token
As mentioned earlier, self-hosted runners need to authenticate against the GitHub REST API, that's where the GitHub App comes in.  
But the private key generated earlier is not passed to the Container App, it is used to generate a short-lived token that we pass to the runner. This is done by the following step in the workflow:
```yaml
- name: Generate access token
  id: generate-access-token
  uses: actions/create-github-app-token@v1
  with:
    app-id: ${{ vars.GH_APP_ID }}
    private-key: ${{ secrets.GH_APP_PRIVATE_KEY }}
    skip-token-revoke: true
``` 
{: file=".github/workflow/provision-runners.yml" }
A few interesting things here:
- The GitHub App id is stored in an action _variable_ (as it's not sensitive), the private key in an action _secret_ (as it's of course sensitive)
- This step uses a verified action from the [marketplace](https://github.com/marketplace/actions/create-github-app-token)
- Note the use of the `skip-token-revoke` property: by default the generated token is automatically revoked at the end of the workflow. We don't want that here as the token will be used after the deployment and we want to avoid [401](https://http.cat/401) errors.

### Run the Bicep deployment from GitHub Actions
```yaml
- name: Bicep deploy
  uses: azure/arm-deploy@v1
  with:
    scope: resourcegroup
    resourceGroupName: ${{ vars.RG_NAME }}
    template: ./infra/02-app/main.bicep
    parameters: >
      project=${{ vars.PROJECT }} 
      acrName=${{ vars.ACR_NAME }} 
      acaEnvName=${{ vars.ACA_ENV_NAME }} 
      imageTag=from-base 
      gitHubAccessToken=${{ steps.generate-access-token.outputs.token }} 
      gitHubOrganization=${{ github.repository_owner }} 
    deploymentName: deploy-aca-gh-runners-app
``` 
{: file=".github/workflow/provision-runners.yml" }

```
resource acaApp 'Microsoft.App/containerApps@2023-05-01' = {
  name: 'ca-${project}'
  location: location
  tags: tags
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${acaMsi.id}': {}
    }
  }
  properties: {
    managedEnvironmentId: acaEnv.id
    configuration: {
      activeRevisionsMode: 'Single'
      registries: [
        {
          server: acr.properties.loginServer
          identity: acaMsi.id
        }
      ]
      secrets: [
        {
          name: 'github-access-token'
          value: gitHubAccessToken
        }
      ]
    }
    template: {
      containers: [
        {
          name: 'github-runner'
          image: '${acr.properties.loginServer}/runners/github/linux:${imageTag}'
          resources: {
            cpu: json(containerCpu)
            memory: containerMemory
          }
          env: [
            {
              name: 'ACCESS_TOKEN'
              secretRef: 'github-access-token'
            }
            {
              name: 'RUNNER_SCOPE'
              value: 'org'
            }
            {
              name: 'ORG_NAME'
              value: gitHubOrganization
            }
            {
              name: 'EPHEMERAL'
              value: '1'
            }
            {
              name: 'RUNNER_NAME_PREFIX'
              value: project
            }
          ]
        }
      ]
    }
  }
}
```
{: file="infra/modules/containerApp.bicep" }


## Wrapping-up