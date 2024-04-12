---
title: Federated credentials between Azure Pipelines and Terraform's Azure DevOps provider
tags: [azure, azure-devops, azure-pipelines, terraform]
img_path: /assets/img/azdo-terraform-oidc
---

Workload identity federation is becoming more and more supported in the Azure ecosystem, and there is already a lot of content on how to use it for deploying Azure resources (I briefly mention it [here]({% post_url 2023-11-27-github-runner-container-app-part1 %}#one-last-word-about-managed-identities) and [there](https://github.com/xmi-cs/aca-gh-actions-runner?tab=readme-ov-file#connect-github-with-azure) in a GitHub Actions context).  
Since last month it's also supported by the Terraform provider for Azure DevOps. As there are less content for this provider, let's see in this post how to configure it and make it work with a simple pipeline.

## Add a service principal user in your organization
The first thing to do is to create an _app registration_ in Entra ID (formerly Azure AD). Just give it a name and keep the other settings as default:  
![Create and App Registration in the portal](/portal-app-registration.png){: width="600"} _This can also be done with tools like the Azure CLI or PowerShell_
Then in Azure DevOps, in _Organization settings_, you can add the _service principal_ just like any normal user (don't forget to add it to at least one project):
![Add service principal in Azure DevOps](/azdo-user.png){: width="600"} _For this sample we will use the Project Contributors group but you might need to use another one depending on your needs_

> Choosing the _Basic_ access level will use consume one licence (the first 5 users are free for each organization) 
{: .prompt-warning }

> Before doing this you need to connect your Azure DevOps organization with your Entra ID tenant, checkout the docs [here](https://learn.microsoft.com/en-us/azure/devops/organizations/accounts/connect-organization-to-azure-ad?view=azure-devops) if you need.
{: .prompt-tip }

## Create a service connection with federated credentials
Now that the service principal has access to a project in Azure DevOps, we need to _federate_ its identity with Azure Pipelines runners. From your project settings in Azure DevOps, create a new service connection, select _Workload Identity federation (manual)_, give it a name, and clic _Next_:  
![Issuer and subject identifier in Azure DevOps](/azdo-sc-issuer.png){: width="350"}

Here you can retrieve the _issuer_ and the _subject identifier_ that you need to set on the service principal. Go back to your app registration in the Azure portal, click on _Certificates & secrets_, _Federated credentials_, _Add credential_, and select the _Other issuer_ scenario. Then you can report the _issuer_ and _subject identifier_ values from Azure DevOps, and give a name to the credential:
![Issuer and subject identifier in Azure portal](/portal-issuer.png){: width="600"}

Back in the service connection creation in Azure DevOps, you have probably noticed that it's saved as a draft, so we need to finalize the set-up. We need to provide the following data:
- An Azure subscription name and id
- The service principal id (you can use the _Application (client) id_ from the app registration's blade)
- The Entra tenant id (also from the same blade in the Azure portal)

> Clicking on _Verify and save_ now will get you an error saying that the service principal (the _client_) doesn't have authorization to perform the action `Microsoft.Resources/subscriptions/read` over the scope of your Azure subscription
{: .prompt-warning }

Before clicking on _Verify and save_, as we are using an _Azure Resource Manager_ service connection, Azure DevOps checks that the service principal has at least read access to the Azure subscription. This seems unneeded as we don't want to interact with Azure here (just with Azure _DevOps_), but as there is no _Azure DevOps_ service connection type, we need to provide an Azure subscription and give access to it.  
So to work around this, simply add a role assignment with the `Reader` role to the service principal on the Azure subscription (documentation [here](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-portal?tabs=delegate-condition) if you need).

Finally, you can click on _Verify and save_ to finalize the service connection set-up, it should be ok now.

## Let's add some code
This is where the fun begins, as the set-up steps are finally done. As a simple example, let's say we want to _data source_ our Azure DevOps project in Terraform and _output_ its id, so the `main.tf` looks like this:
```hcl
data "azuredevops_project" "test" {
  name = var.project_name
}

output "project_id" {
  value = data.azuredevops_project.test.id
}
```
{: file="main.tf" }

Let's focus on the provider configuration, as you can see in the provider's documentation [here](https://registry.terraform.io/providers/microsoft/azuredevops/latest/docs/guides/authenticating_service_principal_using_an_oidc_token), we need to provide a tenant id, a client id, an OIDC token 😱, the URL of the Azure DevOps organization, and specify the `use_oidc` flag.  
There are several ways to provide these values: directly in the code, through environment variables, or text files... I have chosen to only set the `use_oidc` flag in the code to keep it simple:
```hcl
terraform {
  required_providers {
    azuredevops = {
      source  = "microsoft/azuredevops"
      version = ">=1.0"
    }
  }
}

provider "azuredevops" {
  use_oidc = true
}
```
{: file="versions.tf" }

Now on the pipeline side, all we need is an `AzureCLI` task so that we can use the service connection
```yaml
name: TestTerraformAdoProvider

steps:
  - task: AzureCLI@2
    name: RunTerraform
    inputs:
      azureSubscription: sc-azdo-oidc-demo # The name of your service connection goes here
      scriptType: bash
      scriptLocation: inlineScript
      addSpnToEnvironment: true
      inlineScript: |
        export ARM_TENANT_ID="$tenantId"
        export ARM_CLIENT_ID="$servicePrincipalId"
        export ARM_OIDC_TOKEN="$idToken"
        terraform init
        terraform apply -auto-approve
    env:
      AZDO_ORG_SERVICE_URL: $(System.CollectionUri)
      TF_VAR_project_name: $(System.TeamProject)
```
{: file=".azuredevops/pipeline.yml" }

A few things to notice here:
- The `addSpnToEnvironment` input is key: it populates the `$tenantId`, `$servicePrincipalId` and `$idToken` variables in the inline script so we can pass them to Terraform through environment variables
- Predefined System [variables](https://learn.microsoft.com/en-us/azure/devops/pipelines/build/variables?view=azure-devops&tabs=yaml#system-variables-devops-services) are used to get the Azure DevOps organization URL and project name so that we don't need to hard-code them 👌

And that's it, running this pipeline should get a result like this:
![Pipeline run in Azure DevOps](/azdo-pipeline.png){: width="600"}

Which might seem pointless but demonstrates that the _federated_ connection between the Azure Pipelines agents and Azure DevOps works through the Terraform provider 🙌

## Wrapping-up
In this post we have seen how to use the federated connection with the Terraform provider for Azure DevOps. I usually don't do many tutorial posts like this, but as it's way easier to find content about Azure (not _DevOps_), and the documentation is split across several websites, I guess providing a full tutorial is worth it.  
As I have learned a few things about authentication in Azure DevOps, there might be other posts on this fancy topic in the future 🤓