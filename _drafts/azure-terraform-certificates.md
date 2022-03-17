---
title: "Generate TLS certificates with Terraform for your Azure projects"
tags: [terraform, azure, app-service]
img_path: /assets/img/azure-terraform-certificates
---

We all love starting pet projects, so we tend to buy custom domains as it can be fairly cheap. SSL/TLS certificates on the other hand used to be pricey, but today there are several solutions to get these for free.  
And this is for the good cause as every website should be secured by certificates nowadays.  
In this post I will share 3 ways to automate the generation of certificates with Terraform for your Azure projects.


## Presentation of the context

Let's say we have the following architecture as a starting point:  
![Starting architecture](01-http.png) _Let's start with a Web App bound to a custom domain_  
So we have the following components:
- An App Service running in a plan with in the Basic tier at least
- A DNS zone with at least the following records:
  - A CNAME record pointing to the default App Service hostname (`*.azurewebsites.net`)
  - A TXT records to verify the domain ownership
- These two records allow the creation of a custom hostname *binding* at the App Service level

Everything is created using Terraform CLI from my terminal, including the DNS records, that's why I'm using Azure DNS 😎  
With this setup the App Service is accessible on the custom hostname using HTTP only, if we try to access it using HTTPS our browser displays an error as the only certificate it can find is bound the `azurewebsites.net` domain.  
Now that we have our basic setup, let's see how we can secure this web app.


## GitHub repository

Before that, all the code from this post is available in the following [repo](https://github.com/xaviermignot/terraform-certificates).  
The `readme` explains how to get started if you want to create the resources and generate the certificates in your own subscription.


## Level 1: generating a self-signed certificate

As a very first step we will generate a *self-signed* certificate, something we will not do for production but can be handy for a quick test.  
HashiCorp provides a [tls](https://registry.terraform.io/providers/hashicorp/tls/latest) provider to generate the certificate using Terraform just like we will do using openssl in Linux or PowerShell in Windows.  
Here is the HCL code to do this:
```hcl
resource "tls_private_key" "private_key" {
  algorithm = "RSA"
}

resource "tls_self_signed_cert" "self_signed_cert" {
  key_algorithm   = tls_private_key.private_key.algorithm
  private_key_pem = tls_private_key.private_key.private_key_pem

  validity_period_hours = 48

  subject {
    common_name = var.common_name
  }

  allowed_uses = ["key_encipherment", "digital_signature", "server_auth"]
}
```
It's just two resources: a private key and a certificate generated using this private key.  
We could stop here but in an Microsoft context we need to export the certificate in PFX format to use it in an Azure service such as App Service or Application Gateway.  
This is done by combining the use of the HashiCorp [random](https://registry.terraform.io/providers/hashicorp/random/latest) provider (to generate a password) and the third-party provider [pkcs12](https://registry.terraform.io/providers/chilicat/pkcs12/latest):
```hcl
resource "random_password" "self_signed_cert" {
  length  = 24
  special = true
}

resource "pkcs12_from_pem" "self_signed_cert" {
  cert_pem        = tls_self_signed_cert.self_signed_cert.cert_pem
  private_key_pem = tls_private_key.private_key.private_key_pem
  password        = random_password.self_signed_cert.result
}
```
And that's it, the certificate is ready to be used as an App Service certificate as you can see in the full code [here](https://github.com/xaviermignot/terraform-certificates/blob/main/01_self_signed/main.tf). The following diagram also illustrates this first step:
![Generating self-signed certs](02-self-signed.png)_Diagram of the generation of self-signed cert_ 

As you have probably guessed this certificate will not make our browser happy as our machine is not recognized as a trusted authority:

We will fix this in the next step for no additional cost.

## Level 2: requesting a certificate from Let's Encrypt

## Level 3: using a certificate managed by Azure

## Wrapping up