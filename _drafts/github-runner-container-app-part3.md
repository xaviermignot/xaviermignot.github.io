---
title: "Scale your org's GitHub runners in Azure Container Apps"
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

## What next ?

## Wrapping-up