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

## Put the GitHub App key in a Key Vault
Instead of setting the GitHub App key in a secret reference, it's better to store it in a Key Vault as it's super sensitive.  
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

> The snippets in this section highlights what has changed since the previous post, but the whole code is available [here](https://github.com/xmi-cs/aca-gh-actions-runner/blob/main/infra/modules/containerAppJob.bicep).
{: .prompt-info }

## Testing the updated runners
After deploying the new version of the Bicep code, I have tested it just like I did for the [previous post]({% post_url 2023-12-04-github-runner-container-app-part2 %}#testing-the-runner-at-scale).  
It worked out of the box, then I have waited a few hours, then a few days, and it continued to work ! So the problem I was having is definitively gone, and the solution is fully functional 🥳

## What next ?
Now that the main remaining issue is solved, where could the focus go in the future ? Of course there are many improvements to do 🤓

### Automating the generation of the GitHub App key
At first I wanted to automate the generation of the GitHub App Key and put it in the Key Vault using a workflow instead of manually update a GitHub Action secret. It would be a cool _zero-touch_ approach but it turns out there is no endpoint in the REST API for generating an App key.  

The closest way would be to implement the GitHub App [manifest flow](https://docs.github.com/en/apps/sharing-github-apps/registering-a-github-app-from-a-manifest) but it requires human interaction so definitively not suitable for workflows.  

So I'll stick with the Action secret method for now.

### Use the builtin token for updating variables
Another thing I would like to remove is the need to generate a token in the `deploy-prerequisites` workflow:
```yaml
- name: Generate access token
  id: generate-access-token
  uses: actions/create-github-app-token@v1
  with:
    app-id: ${{ vars.GH_APP_ID }}
    private-key: ${{ secrets.GH_APP_PRIVATE_KEY }}

- name: Update GitHub variables
  run: |
    # (Sets a bunch of other variables like this one)
    gh variable set GH_APP_PRIVATE_KEY_SECRET_URI --body ${{ steps.bicep-deploy.outputs.gitHubAppKeySecretUri }}          
  env:
    GITHUB_TOKEN: ${{ steps.generate-access-token.outputs.token }}
```
{: file=".github/workflows/deploy-prerequisites.yml" }
Initially I needed to generate access tokens for the Container Apps, so after letting the runners access the App key I though I could remove the token generation in the Action workflow, but no.  

The solution is deployed by two workflows with dependencies, so to "pass" values between the workflows I use GitHub Action variables using the GitHub CLI. The GitHub CLI needs an access token, I could use the builtin `GITHUB_TOKEN` for that but at the time of writing this it's not possible to grant to the `GITHUB_TOKEN` the [permission](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#permissions-for-the-github_token) to read or write these variables, so I have to keep generating access token for now.  

I guess this will be updated in the near future (the Action variables are still in preview), so I will be able to change this later.

### Making the solution more private
One of the main advantage of self-hosted runners is the ability to put them in your environment and not publicly accessible on the Internet.  
Currently my implementation doesn't do that, it could be improved by putting the Container App in a VNET, accessing the Container Registry and the Key Vault through private endpoints, and so on.  
Honestly I don't plan to add this for now, my intention was to share a simple baseline for hosting GitHub runners in Container Apps. Integrating it in a private environment should not be hard to do like for any Azure resource.

### Improving the scaling rules (and the performance)

### Manage several labels

## Wrapping-up