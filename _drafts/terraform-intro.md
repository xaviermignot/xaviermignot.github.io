---
title: Introduction to Terraform
tags: [terraform]
---

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

### Terraform providers

## What is Terraform Cloud ?
Terraform Cloud is an hosted service provided by HashiCorp. It provides the following main features:
- Storing the state of the infrastructure securely and remotely
- Providing *agents* to run Terraform commands remotely
- Helping teams to work together as everything happens remotely and not on someone's machine

As Terraform is [open-source](https://github.com/hashicorp/terraform) but provided by a [corporation](https://www.hashicorp.com/), it's important to know how this corporation earns money. At first I was thinking that Terraform was paid but it turns out it's free for teams up to 5 users 😎  
Paid plans starts for bigger teams and there is also a self-hosted version of the same application called Terraform Enterprise. So that's how HashiCorp earns money, with big companies.  
And you can get started for free, which is pretty nice, and pretty clever too 😏
