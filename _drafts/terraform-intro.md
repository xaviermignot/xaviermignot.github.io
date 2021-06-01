---
title: Manage your pet projects resources using Terraform Cloud
tags: [terraform]
---

Over the last few years I have been interested by IaC (Infrastructure as Code), but never had the opportunity to use it in my professional life. For my few pet projects I was lazy so I have created the resources using the Azure portal, click click mode FTW !  
Finally for my last project I really wanted to get into Terraform so I made the effort to start using it, and overall I was happy to find out that I can use Terraform Cloud for free, so I have decided to build my next few projects with it, with a method I will share in this post.


## Basics - What is TF & TF Cloud

This post is not a deep dive into Terraform, we will stick to the basics. But if you are very new to Terraform, I encourage you to browse the product [website](https://www.terraform.io/) and follow a [basic tutorial](https://learn.hashicorp.com/terraform) as it's the best way to know what the tool is about.  
If you still need a little introduction, let's list a few definitions before getting to the point.

### Terraform
Terraform is an *infrastructure as code* (IaC) tool to manage infrastructure in an automated way. It's open source, free to use and works with many *providers* such as AWS, Azure, Kubernetes, and more.  
The use of Terraform is centered around theses four main commands:
```terminal
# Initialize your workspace
$ terraform init
# Calculate the changes to make to your infrastructure
$ terraform plan
# Apply the previously calculated changes
$ terraform apply
# Destroy the infrastructure :O
$ terraform destroy
```

### HashiCorp Configuration Language (HCL)
When working with Terraform, the infrastructure is described using code written in *HCL* for *HashiCorp* (the company behind Terraform) *Configuration Language*.  

### Terraform State and Backend
One important thing to mention about Terraform is that it uses a *declarative* approach, in opposition to an *imperative* approach. So the Terraform code describes what the infrastructure should be, and not which actions should be performed against the infrastructure.  
So basically you describe what you want, and Terraform guesses the actions to perform on its own.  
To do that Terraform maintains a *state* which is basically a JSON representation of you infrastructure.  
Where the state is stored depends on the *backend* (another important concept of Terraform) you choose, there a several possibilities:
- In a file on your machine if you're working locally
- In some cloud storage like Azure Storage or AWS S3
- In Terraform Cloud, aka the platform provided by HashiCorp to store the versions of your state and run Terraform commands from disposable virtual machines  

The *local* backend is recommended for getting started, but once you're getting comfortable with the tooling you can move to Terraform cloud, even for pet projects as it's free to use for teams up to 5 users.

### Terraform Cloud

### Terraform providers

## Using Terraform Cloud from your terminal

## Organization of the repo
### File organization
### Do not commit tf-backend.tf file

## Wrapping up