---
title: "Terraform vs Bicep: the differences you should really know"
tags: [azure, bicep, terraform]
img_path: /assets/img/terraform-vs-bicep
image:
  path: banner.png
---

For a little bit more than a year my professional coding time has been dedicated to writing Infrastructure-as-Code and CI/CD pipelines for several customers working with Azure. I now have some real-world experience in both Terraform and Bicep and as I have read several posts comparing the two, I want to share my opinion on this.  
If there is one thing to retain it would be this: this is a not a mono or multi cloud issue, you have to think beyond that. To elaborate on this, this post will focus on two differences that are not talked about enough.

## First difference: the execution mode
One of the most important thing to understand is what is happening on the machine executing Terraform or Bicep (note that the machine could be your workstation or a CI/CD runner).  

### Using Bicep
When you give a file to Bicep, it will find the dependant Bicep files, "compile" the whole thing as an ARM template and submit a _deployment_ in a single API call to Azure. From there all the work is done within Azure, the caller (your workstation or CI/CD runner) monitors the status of the deployment and waits for its completion.  
You can also monitor the execution and see the result from the Azure portal in the deployment blade of the targeted management group, subscription or resource group.  
![Bicep execution sad Escobar meme](01-execution-mode-bicep.jpg){: width="400"}

### Using Terraform
When you run `terraform plan` from a folder, it compares the whole _configuration_ (the `*.tf` files) with the _state_ to determine the changes to make on the resources in Azure. Then running `terraform apply` will make the changes on the resources and reflect them in the state.  
For both commands there is no _deployment_ submitted to Azure, but a lot of calls to the management API are made to create/read/update/delete the resources in Azure. The order of the API calls is determined by the logic of Terraform and its providers, this logic is executed on the machine running the Terraform CLI (your workstation or CI/CD runner).  
Once the configuration has been applied you won't see any _deployment_ in the Azure portal, as the AzureRM provider of Terraform doesn't use them. Instead you can see a bunch of entries in the _Activity log_ of the affected resources.  
![Terraform execution conspiracy whiteboard meme](02-execution-mode-terraform.jpg){: width="400"}

### What does it change ?

#### The ability to interact with other tools, APIs, or clouds

#### Code organization

#### The need to determine values at the start of deployment

## Second difference: state vs no state
The state is a key-concept specific to Terraform. Basically it's an abstract layer used to map real world resources to the configuration. It's something you might dislike at first for the following reasons:
- As it contains a representation of you infrastructure, including sensitive data, it's something that you need to secure. Whether you put it in a storage account, use Terraform Cloud or another _backend_, this is an important governance choice to make, and it can be complicated depending on you organization
- If a change is made without Terraform on a resource previously created with Terraform, it will introduce a _drift_, aka a difference between the state and the real world infrastructure. This can be made by humans using the portal, but also by Azure itself: for instance when a managed certificated is automatically renewed by Azure, the new thumbprint is not updated in Terraform's state.  
Depending on the situation it can be trivial or tricky to address, but you should be prepared as it will very likely happen.

But once familiar with the state you'll become confident in dealing these and you'll appreciate the features it brings.
Of course in Bicep, there is no state, we can say that the infrastructure is the sate, which is simpler to handle at first but you will see in the rest of this post the features you might miss.  
![No state no drift meme](03-no-state.jpg){: width="400"}

### What does it change ?

#### Destroying resources

#### Behavior regarding outside changes 

#### Ease of refactoring

#### Storing non-resources stuff

## Other minor differences

### Language features

### VS Code integration

### Tooling

## Some statements to debunk

### "Bicep always gets new features support first"

## Wrapping up