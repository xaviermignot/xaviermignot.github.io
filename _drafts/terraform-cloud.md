---
title: Manage your pet projects resources using Terraform Cloud
tags: [terraform]
img_dir: /assets/img/terraform-cloud
---

Over the last few years I have been interested by IaC (Infrastructure as Code), but never had the opportunity to use it in my professional life. For my few pet projects I was lazy so I have created the resources using the Azure portal, click click mode FTW !  
Finally for my last project I really wanted to get into Terraform so I made the effort to start using it, and overall I was happy to find out that I can use Terraform Cloud for free, so I have decided to build my next few projects with it, with a method I will share in this post.


## What is Terraform and Terraform Cloud ?

This post is not a deep dive into Terraform, we will stick to the basics. But if you are very new to Terraform, I encourage you to browse the product [website](https://www.terraform.io/) and follow a [basic tutorial](https://learn.hashicorp.com/terraform) as it's the best way to know what the tool is about.  
To better understand the differences between Terraform and Terraform Cloud, let's sum up the differences real quick:
- Terraform is the open-source CLI tool with the `plan`, `apply`, `destroy` commands that can run anywhere (including on your machine or in Terraform Cloud)
- Terraform Cloud is a SaaS offering to manage Terraform runs and states remotely

### Is it free ?
Terraform CLI is [open-source](https://github.com/hashicorp/terraform) and free to use. Terraform Cloud is also free for teams up to 5 users, so it's perfect for pet projects.  
Terraform Cloud has paid plans for larger organizations, and also a self-hosted version called Terraform Enterprise. That's not the topic of this post, but it's important to know how the company behind the tool makes money.


## Let's get started !

Let's get finally to the point, shall we ? For this sample we will create an Azure Function App which will result in creating the following resources:
- A resource group
- An App Service Plan
- A storage account
- A Function App
- An Application Insights account (optional but always handy to monitor the app's telemetry)

### Prerequisites
To do this we will need the following prerequisites:
- A Terraform Cloud account with an organization, to create one follow the guidelines from [here](https://learn.hashicorp.com/tutorials/terraform/cloud-sign-up?in=terraform/cloud-get-started#create-an-account)
- An Azure subscription, a [free](https://azure.microsoft.com/en-us/free/) one will be enough
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/) installed and connected to your subscription

### Create your workspace
Once you have your *organization* (which represents your *team*) in Terraform Cloud, you will need a *workspace* (which represents your *project*).  
Go to [Terraform Cloud](https://app.terraform.io/), select you organization, and click on the *"__New workspace__"* button.  

On the next page you have to choose a workflow, choose *CLI-driven workflow*:  
![CLI-driven workflow]({{ page.img_dir }}/01-workspace-type.png)
_The CLI-driven workflow on the new workspace page_  
Using this workflow allows you to run Terraform commands from your machine, but they will run remotely in Terraform Cloud. It's the best of two worlds approach as you get the simplicity of running the commands from your terminal of choice, and you get the security of having the state stored remotely, and the runs are accessible in Terraform Cloud.

### Give Terraform Cloud access to your Azure subscription

### Get the Terraform code

## Organization of the repo
### File organization
### Do not commit tf-backend.tf file

## Wrapping up