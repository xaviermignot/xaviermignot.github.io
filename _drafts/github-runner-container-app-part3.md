---
title: "Follow-up on GitHub runners in Azure Container Apps"
tags: [github-actions, azure, container-apps, bicep]
img_path: /assets/img/gh-runner-aca-part-3
image:
  path: banner.png
---

I have recently gone back into the code of my series about GitHub self-hosted runners and Azure Container Apps. I have made enough adjustments to add a third post to the series.  

## In the previous episodes...
The first post of the series covers the essentials of the solution, and the second one adds scaling with KEDA and switches to Container Apps Jobs. You can check them out if you haven't already.  
> There is also a GitHub [repository](https://github.com/xmi-cs/aca-gh-actions-runner) with all the code and instructions on how to deploy it in your environment.
{: .prompt-info }
At the end of the second post, there still was an issue with authentication. It works using a token generated using a GitHub App, and once expired I had to manually run a workflow to generate a fresh token.  

## Updates on the KEDA scaler
The first good news is the version of KEDA which has been updated to `2.12` on my Container App Environment:
![KEDA version in the Azure portal](portal-environment.png)_KEDA version went from 2.10 to 2.12 🚀_

This is a big deal as authentication using a GitHub App key was introduced in the `2.11` version of the scaler. In the previous post I still had to use a token for authentication because of the `2.10` version of the scaler.

## Put the GitHub App Key in a Key Vault
Instead of setting the GitHub App Key in a secret reference, it's better to store it in a Key Vault as it's super sensitive.  
This is very easy to do using Bicep, the only thing to notice is that even if it's a _key_, we have to store it in a Key Vault _secret_ so the Container App Job can reference it.  

> RBAC has also been updated to grant the _Secret User_ role on the Key Vault to the Managed Identity linked to the Container App Job.
{: .prompt-info }

## Update the Container App Job to use the Key Vault
First, the secret containing the access token is replaced by a reference to the new secret in the Key Vault:
```
resource acaJob 'Microsoft.App/jobs@2023-05-01' = {
  properties: {
    configuration: {
      secrets: [
        {
          name: 'github-app-key'
          keyVaultUrl: gitHubAppKeySecretUri
          identity: acaMsi.id
        }
      ]
    }
  }
}
```
{: file="infra/modules/containerAppJob.bicep" }

The secret is used in two places by the Container App Job, as an environment variable:
```
resource acaJob 'Microsoft.App/jobs@2023-05-01' = {
  properties: {
    template: {
      containers: [
        {
          name: 'github-runner'
          env: [
            {
              name: 'APP_ID'
              value: gitHubAppId
            }
            {
              name: 'APP_PRIVATE_KEY'
              secretRef: 'github-app-key'
            }
          ]
        }
      ]
    }
  }
}
```
{: file="infra/modules/containerAppJob.bicep" }
Note that two variables are used, one for the GitHub App id, one for the key. These are used by the container itself to authenticate against the GitHub REST API. 

On the scaling side, the authentication parameter `appKey` references the same secret as the environment variable. Following the KEDA [documentation](https://keda.sh/docs/latest/scalers/github-runner/), using the key for authentication requires to add the application id and installation id in the `metadata`: 
```
resource acaJob 'Microsoft.App/jobs@2023-05-01' = {
  properties: {
    configuration: {
      triggerType: 'Event'
      eventTriggerConfig: {
        scale: {
          rules: [
            {
              name: 'github-runner-scaling-rule'
              type: 'github-runner'
              auth: [
                {
                  triggerParameter: 'appKey'
                  secretRef: 'github-app-key'
                }
              ]
              metadata: {
                owner: gitHubOrganization
                runnerScope: 'org'
                applicationID: gitHubAppId
                installationID: gitHubAppInstallationId
              }
            }
          ]
        }
      }
    }
  }
}
```
{: file="infra/modules/containerAppJob.bicep" }

The snippets in this section highlights what has changed since the previous post, but the whole code is available [here](https://github.com/xmi-cs/aca-gh-actions-runner/blob/main/infra/modules/containerAppJob.bicep).

## What next ?

## Wrapping-up