---
title: "Scale your org's GitHub runners in Azure Container Apps"
tags: [github-actions, azure, container-apps, bicep]
img_path: /assets/img/gh-runner-aca-part-2
image:
  path: banner.png
---

This is the second post of the series about hosting your GitHub org's self-hosted runners in Azure Container Apps. The [first episode]({% post_url 2023-11-27-github-runner-container-app-part1 %}) covers the essentials of the solution, concluding with a testing workflow running in an Azure Container App.  
This post will focus on _scaling_ the self-hosted runners, using the builtin KEDA integration of Azure Container Apps.

## In the previous episode...
If you haven't read my previous post you can check out [this section]({% post_url 2023-11-27-github-runner-container-app-part1 %}#github-repository-and-organization) that presents the solution used in this series.  
> There is also a GitHub [repository](https://github.com/xmi-cs/aca-gh-actions-runner) with all the code and instructions on how to deploy it in your environment.
{: .prompt-info }

## The KEDA GitHub runner scaler
[KEDA](https://keda.sh/) is the de-facto autoscaler of the Kubernetes ecosystem. It can sale workloads in a cluster based on event-driven sources called _scalers_. Among the many available scalers there is the [GitHub Runner Scaler](https://keda.sh/docs/latest/scalers/github-runner/), and like any KEDA scaler, we can use it to scale our Container App.  
It works by querying the GitHub REST API to get the number of queued jobs, and will add or remove _replicas_ to the Container App accordingly. This means we need to configure how the scaler will authenticate against the GitHub REST API, and which filters to count the jobs that are worth a scale.  

### Choosing the right version of the scaler
The link to the KEDA documentation in the previous paragraph leads to the latest version of KEDA (`2.12` at the time of writing this post). Before browsing the docs I strongly advise you to check the KEDA version used by your Container Apps Environment:
![KEDA version in the Azure portal](/01-portal-environment.png)_Looks like I don't have the latest version 🥹_

Azure Container Apps is a _managed_ service where you don't manage the KEDA version used by the underneath cluster. Using Azure Kubernetes Service will probably give you more control on this, but with Container Apps the updates are rolled by service.  

Before I understood this, I was trying to use the scaler's authentication with a GitHub App private key, which is not available in the KEDA version used by my Container App Environment. 

## About ephemeral runners
GitHub [recommends](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/autoscaling-with-self-hosted-runners#using-ephemeral-runners-for-autoscaling) the use of _ephemeral_ runners for autoscaling. It means that each runner processes a single job, and each job is processed by a fresh runner. This makes job execution more predictable and prevent the risk of using a compromised runner or exposing sensitive information.  

The use of ephemeral runners depends on the container image we use. The [image]({% post_url 2023-11-27-github-runner-container-app-part1 %}#create-a-dockerfile-or-use-another-one-as-a-base) used in the previous post requires the `EPHEMERAL` environment variable set to `1`.

## Container Apps Jobs
In the previous post a Container App has been set-up a single self-hosted runner. The App was always running, regularly polling the GitHub REST API for jobs to run, and the same runner was processing all the jobs.  
At first I wanted to add a scaling rule on top of that but after some research there is a better approach by using Azure Container Apps [Jobs](https://learn.microsoft.com/en-us/azure/container-apps/jobs).  

Jobs are designed for containerized tasks with a finite duration just like our ephemeral runners.  
They also support event triggers from KEDA scalers so it's definitively a good fit for this scenario.

Before jumping into some code, this little diagram show what will happen once the runners will scale:
![Diagram](/02-diagram.png)_It also shows the changes from the diagram of the previous post_

## Update the Bicep code

### Turning the Container App into a job
To use a Container App Job, we need to use the `Microsoft.App/jobs` resource type instead of `Microsoft.App/containerApps`. This change is easy as both resource types share a lot of common properties:
- The `registries` and `secrets` declaration are the same
- The `template/containers` array is also the same for both resources

There is a slight difference on the property that links to the Container App Environment (`environmentId` vs `managedEnvironmentId`), and the scaling part is also different so let's focus on that.

### Configuring the scaler
This is the most important part, it consists in matching the [trigger specification](https://keda.sh/docs/2.10/scalers/github-runner/#trigger-specification) from the scaler's docs with a [rule](https://learn.microsoft.com/en-us/azure/templates/microsoft.app/jobs?pivots=deployment-language-bicep#jobscalerule) in the `eventTriggerConfig` object of the resource:
- the `metadata` object contains the parameters of the scaler: the scope, organization, labels, etc.
- the `auth` array tells how the scaler authenticates against the GitHub REST API: using a secret reference to a PAT in our case as the GitHub App private key is not supported in our version of KEDA.

This code snippet highlights the required changes:
```
resource acaJob 'Microsoft.App/jobs@2023-05-01' = {
  // ...
  properties: {
    environmentId: acaEnv.id
    configuration: {
      // ...
      replicaTimeout: 1800
      triggerType: 'Event'
      eventTriggerConfig: {
        scale: {
          rules: [
            {
              name: 'github-runner-scaling-rule'
              type: 'github-runner'
              auth: [
                {
                  triggerParameter: 'personalAccessToken'
                  secretRef: 'github-access-token'
                }
              ]
              metadata: {
                owner: gitHubOrganization
                runnerScope: 'org'
              }
            }
          ]
        }
      }
    }
    // ...
}
```
{: file="infra/modules/containerAppJob.bicep" }

> The whole code is available in the GitHub repo, by default the Container App Job [module](https://github.com/xmi-cs/aca-gh-actions-runner/blob/feat/add-scaling/infra/modules/containerAppJob.bicep) is selected, you can still check/compare with the Container App [module](https://github.com/xmi-cs/aca-gh-actions-runner/blob/main/infra/modules/containerApp.bicep).
{: .prompt-info }

## Testing the runner at scale
Once a deployment has been made with the new version of the Bicep code, no runner should be visible at first in GitHub. But if we manually trigger the `Test self-hosted runners` workflow several times, we should see this:
![Active runners](/03-runners.png)_The jobs are being processed in parallel_

In the Azure portal, we can see the triggered jobs in the _Execution history_ blade:
![Execution history](/04-portal-execution-history.png)_This is where we can monitor current and past runs_
From here we can also troubleshoot in the logs, sadly the _Log stream_ blade is not (yet ?) available for jobs so we have to run queries in Log Analytics for that.

Once all the jobs have finished, everything is marked as _completed_ in the Azure portal, and no runner appear in the GitHub UI as the KEDA scaler have scaled them to 0.

> I have kept the default scaling parameters (0 to 10 instances). This can be changed and even combined with other rules like a [CRON](https://keda.sh/docs/latest/scalers/cron) one if you want to keep at least one instance alive during the day for faster startup and usage optimization
{: .prompt-tip }

## One last word about GitHub tokens
At the time of publishing this post, to be honest I still have an issue regarding authentication against the GitHub REST API: after a few hours, the scaler gets 401 errors and the runners don't register.  
To workaround this I re-deploy the runners to get a fresh GitHub installation token (I could also use a [schedule](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#schedule)).

I have tried several ways to fix this, without success for now. There is an [optional feature](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/refreshing-user-access-tokens#configuring-your-app-to-use-user-access-tokens-that-expire) in GitHub Apps regarding expiration that I have disabled, but it didn't make any change (it might target user tokens only, I am using installation tokens).  

I guess another way will be to pass the GitHub App private key to the container and let it generate its tokens. The image I'm using supports that but not the scaler yet so I still need to generate tokens.  
And of course I won't do that without putting the key in an Azure Key Vault. This could be an opportunity to automate this part, maybe in another post 😏

If you have lead for me please let me know in the comments 🤗

## Wrapping-up