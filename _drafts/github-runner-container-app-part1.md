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

> If you need an introduction to Azure Container Apps, the best place is the official [documentation](https://learn.microsoft.com/en-us/azure/container-apps). The GitHub doc about self-hosted runners is [here](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners) if you need.
{: .prompt-tip }

## GitHub repository and organization
For the purpose of this series I have created my very own GitHub organization (for free, you can [create](https://github.com/account/organizations/new) your own too !), and carefully crafted [a repository](https://github.com/xmi-cs/aca-gh-actions-runner). I will show only the most important snippets of the code in the post, you can clone/fork the repo to see the whole thing.  
The repo contains a Dockerfile, GitHub workflows and Bicep code to create the necessary resource in Azure, and run a "hello world" Azure CLI script on the self-hosted runners.

## Create a GitHub App
Self-hosted runners use the GitHub REST API to register themselves and query to jobs they have to run. For organizations, the recommended authentication method is to use a GitHub App: the workflows will use a private key to generate tokens to use to authenticate against the REST API.  

From your organization settings, click on _Developer Settings_, then _GitHub Apps_ and _New GitHub App_. Give it a name and a homepage URL (any URL will work), and disable the _Webhook_ feature.  
The important settings are the permissions, set them as follow:
- In _Repository permissions_:
  - Set `Actions` to `Read-only`
  - Set `Metadata` to `Read-only` (it should be selected by default)
  - Set `Variables` to `Read and write`. This is not required for the runners but for one of my workflows
- In _Organization permissions_:
  - Set `Self-hosted runners` to `Read and write`

Keep the other settings as default and click on _Create GitHub App_. On the next page you are prompted to generate a private key: do this and your browser will download a `.pem` file. Note also the id of your app as you will need it later.  

## Create a Dockerfile (or use another one as a base)
The first thing that surprised me when I started working on this, is that there is no official Docker image provided by GitHub.  
You can either create you own Dockerfile from an OS base (as I first did [here](https://github.com/xmi-cs/aca-gh-actions-runner/blob/main/src/Dockerfile.from-ubuntu)), or use a third-party as a base like one from this excellent [repo](https://github.com/myoung34/docker-github-actions-runner). I choose the second option so my Dockerfile is very simple:
```dockerfile
FROM myoung34/github-runner:ubuntu-jammy

# install Azure CLI
RUN curl -sL https://aka.ms/InstallAzureCLIDeb | bash
```

## Deploy the prerequisites
You need a few resources to start:
- A container registry to store your container images
- A Container Apps [environment](https://learn.microsoft.com/en-us/azure/container-apps/environment), to define the pricing, networking and logging settings of one or several Container Apps (one in our case)
- A Log Analytics Workspace to send your logs to. This is optional but very handy in case of troubleshooting

You can create these from the portal, or fork my repo and follow the instructions in the README to connect your fork to your Azure subscription. Then you can launch the `Deploy prerequisites` workflow to automatically create the resources. The workflow also builds and pushes the container image to the registry.


## Provision the runners using Bicep



## Wrapping-up