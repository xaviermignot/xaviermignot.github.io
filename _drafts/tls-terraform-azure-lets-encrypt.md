---
title: "TLS with Terraform and Azure: get certificates from Let's Encrypt"
tags: [terraform, azure, app-service]
img_path: /assets/img/azure-terraform-certificates
---

Following my [previous post]({% post_url 2022-03-18-tls-terraform-azure-self-signed %}) on generating self-signed certificates with Terraform, this one is the second post of the series.  
This time we are going to use Let's Encrypt as the certificate authority (CA) instead of our own machine. As a result we will get trusted certificates that can be used in production, for free.


## Previously in the "TLS with Terraform and Azure" series...

If you haven't read my previous post you can check out [this section]({% post_url 2022-03-18-tls-terraform-azure-self-signed %}/#presentation-of-the-context-for-the-whole-series) to get the common context for the whole series.  
> There is also a GitHub [repository](https://github.com/xaviermignot/terraform-certificates) that contains all the code from this series of posts.
{: .prompt-info }


## Level 2: requesting a certificate from Let's Encrypt

If you don't know what Let's Encrypt is, in a few words it's a free, automated and open certificate authority (CA), allowing everyone to get certificates trusted by browsers at no charge.  
As the Let's Encrypt certificates are valid for 90 days (instead of 1 or years for commercial authorities), automation is strongly recommended.  
So how can we automate this with Terraform ? Using the third-party provider [ACME](https://registry.terraform.io/providers/vancluever/acme/latest/docs). ACME stands for Automated Certificate Management Environment, the protocol used by Let's Encrypt. The Terraform ACME provider supports any ACME CA, so we need to configure Let's Encrypt's endpoint in the provider configuration:
```hcl
terraform {
  required_providers {
    acme = {
      source  = "vancluever/acme"
      version = "~> 2.0"
    }
  }
}

provider "acme" {
  server_url = "https://acme-v02.api.letsencrypt.org/directory"
}
```

> Let's Encrypts provides a staging endpoint you should use for testing: `https://acme-staging-v02.api.letsencrypt.org/directory`  
There is a limit of 50 certificates per domain and per week on the production environment, on the staging it's 30,000 (but the certificates will not be trusted by the browsers)
{: .prompt-tip }  

Once the provider properly configured, we can start creating resources. As for self-signed certificates, it starts here with a private key, and using this private key and an email we create a *registration* which is an account on the ACME server:
```hcl
# Creates a private key in PEM format
resource "tls_private_key" "private_key" {
  algorithm = "RSA"
}

# Creates an account on the ACME server using the private key and an email
resource "acme_registration" "reg" {
  account_key_pem = tls_private_key.private_key.private_key_pem
  email_address   = var.email
}
```

Then we can request a certificate using our account, this is where the real magic happens:
```hcl
# As the certificate will be generated in PFX a password is required
resource "random_password" "cert" {
  length  = 24
  special = true
}

# Gets a certificate from the ACME server
resource "acme_certificate" "cert" {
  account_key_pem          = acme_registration.reg.account_key_pem
  common_name              = var.common_name # The hostname goes here
  certificate_p12_password = random_password.cert.result

  dns_challenge {
    # Many providers are supported for the DNS challenge, we are using Azure DNS here
    provider = "azure"

    config = {
      # Some arguments are passed here but it's not enough to let the provider access the zone in Azure DNS.
      # Other arguments (tenant id, subscription id, and cient id/secret) must be set through environment variables.
      AZURE_RESOURCE_GROUP = var.dns.zone_rg_name
      AZURE_ZONE_NAME      = var.dns.zone_name
      AZURE_TTL            = 300
    }
  }
}
```
The key part is in the `dns_challenge` block of the `acme_certificate` resource.  

Before generating a certificate for our domain, Let's Encrypt checks that we *own* that domain. To do that it proceeds with a DNS *challenge*, basically it generates a random string and will not generate the certificate unless that random string is in a specific TXT record of the DNS zone.  

Obviously the ACME provider does that for us, we just need to let it access our DNS zone. The provider supports *many* DNS providers, in this case we are using Azure DNS.  

So we need to tell the ACME provider *where* our DNS zone is, so we specify its name and resource group name in the `config` block above. Unfortunately the provider cannot use the `az cli` authentication so we to set additional arguments for authentication, so that the provider knows the tenant and the subscription we are using, and a client id and a secret to access to it.  

This is the kind of information that should not be pushed to a git repository, so I prefer to put these as environment variables in a `.env` file that is git-ignored like this:
```sh
export ARM_TENANT_ID="<YOUR AZURE TENAND ID (a guid)>"
export ARM_SUBSCRIPTION_ID="<YOUR AZURE SUBSCRIPTION ID (another guid)>"
export ARM_CLIENT_ID="<AN APP REGISTRATION ID (yep it's a guid too)>"
export ARM_CLIENT_SECRET="<THE APP REGISTRATION SECRET (not a guid this time)>"
```
Then I use the `source .env` command and the ACME provider can use the environment variables to authenticate to Azure and add/remove records in my DNS zone.  

> If you are using Terraform Cloud or running Terraform in a CI/CD context you will not need to do this as the environment variables are already set
{: .prompt-tip }  

Once the [full code](https://github.com/xaviermignot/terraform-certificates/blob/main/02_acme/main.tf) has been applied you can check out the Activity Log of your DNS zone.  
You should see that the service principal corresponding to the environment variables has created and deleted a TXT record in the zone :
![Activity Log](/03-lets-encrypt-activity-log.png) _The creation and deletion of the TXT record should occur within a minute_

And more importantly you can browse your App Service from its custom hostname and see how your browser enjoys this fresh trusted certificate:  
![Browser](/03-lets-encrypt-browser.png){: width="480" } _No more browser warning, you can use this certificate in production_


## Wrapping up

![Diagram](/03-lets-encrypt.png) _The certificate is generated by an external CA, then pushed to the App Service_