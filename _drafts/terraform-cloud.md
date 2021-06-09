---
title: Manage your pet projects resources using Terraform Cloud
tags: [terraform]
---

Over the last few years I have been interested by IaC (Infrastructure as Code), but never had the opportunity to use it in my professional life. For my few pet projects I was lazy so I have created the resources using the Azure portal, click click mode FTW !  
Finally for my last project I really wanted to get into Terraform so I made the effort to start using it, and overall I was happy to find out that I can use Terraform Cloud for free, so I have decided to build my next few projects with it, with a method I will share in this post.


## What is Terraform and Terraform Cloud ?

This post is not a deep dive into Terraform, we will stick to the basics. But if you are very new to Terraform, I encourage you to browse the product [website](https://www.terraform.io/) and follow a [basic tutorial](https://learn.hashicorp.com/terraform) as it's the best way to know what the tool is about.  
Let's sum up the differences real quick:
- Terraform is the open-source CLI tool with the `plan`, `apply`, `destroy` commands that can run anywhere (including on your machine or in Terraform Cloud)
- Terraform Cloud is a SaaS offering to manage Terraform runs and states remotely

### Is it free ?
Terraform CLI is [open-source](https://github.com/hashicorp/terraform) so it's obviously free. Terraform Cloud is also free for teams up to 5 users, so it's perfect for pet projects.  
Terraform Cloud has paid plans for larger organizations, and also a self-hosted version called Terraform Enterprise. That's not the topic of this post, but it's important to know how the company behind the tool earns money.


## Let's get started !

## Using Terraform Cloud from your terminal

## Organization of the repo
### File organization
### Do not commit tf-backend.tf file

## Wrapping up