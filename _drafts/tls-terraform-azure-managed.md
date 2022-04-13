---
title: "TLS with Terraform and Azure: use managed certificates"
tags: [terraform, azure, app-service]
img_path: /assets/img/azure-terraform-certificates
---

This post is the last one of my series on the generation of TLS certificates with Terraform for Azure, after the post about [self signed certificates]({% post_url 2022-03-18-tls-terraform-azure-self-signed %}) and the one about [Let's Encrypt]({% post_url 2022-03-28-tls-terraform-azure-lets-encrypt %}).  
For this one we are going to let Azure *manage* everything by using *managed certificates*, a feature available on several services that let Azure handle the generation and the renewal of certificates.


## Previously in the "TLS with Terraform and Azure" series...

If you haven't read the previous posts of the series you can check out [this section]({% post_url 2022-03-18-tls-terraform-azure-self-signed %}/#presentation-of-the-context-for-the-whole-series) of the first one to get the common context for the whole series.  
> There is also a GitHub [repository](https://github.com/xaviermignot/terraform-certificates) that contains all the code from this series of posts.
{: .prompt-info }  

To give some context very quickly, let's say we have used the `app_service` module of the GitHub repo to create an App Service bound to a custom domain, and now we need to secure this custom domain using a certificate.  
We have already done this using a self-signed certificate, and with a Let's Encrypt certificate, now we are doing it using a managed one.


## Level 3: let Azure handle the certificate stuff

The idea behind managed certificates is pretty simple: if you can prove you *own* a domain, then Azure will issue a certificate for you, valid for this domain.  
And then you don't have to worry about it, Azure will renew the certificate for you, you can forget about it and focus on other things, like your business for instance...

On the diagram side, things looks way simpler than for the other "levels":  
![Diagram](/04-managed.png) _The diagram looks almost the same as for self-signed certificates_

On the code side, we have previously bound the App Service to a custom domain using a `azurerm_app_service_custom_hostname_binding` resource in the `app_service` module:
```hcl
# Bind app service to custom domain
resource "azurerm_app_service_custom_hostname_binding" "app" {
  hostname            = "tf-certs-demo.${var.dns.zone_name}"
  app_service_name    = azurerm_app_service.app.name
  resource_group_name = azurerm_resource_group.rg.name

  depends_on = [azurerm_dns_txt_record.app]
}
```
In the `managed` module, creating the managed is done using the `azurerm_app_service_managed_certificate` with only the *binding id* as an argument:
```hcl
resource "azurerm_app_service_managed_certificate" "managed" {
  custom_hostname_binding_id = var.custom_domain_binding_id
}
```
And finally, back in the `main` module, the `azurerm_app_service_certificate_binding` resource brings everything together by binding the managed certificate to the custom domain:
```hcl
resource "azurerm_app_service_certificate_binding" "cert_binding" {
  certificate_id      = module.managed.certificate_id
  hostname_binding_id = module.app_service.custom_domain_binding_id
  ssl_state           = "SniEnabled"
}
```
Note that comparing to the other posts of the series, I have detailed a little bit more what's happening in the code before and after the generation of the certificate. Otherwise I would have only 3 lines of code show 😬  

> The code snippets above are spread into 3 modules from the repo of the whole series.  
I had previously made another [repo](https://github.com/xaviermignot/app-service-managed-certificate-demo) dedicated to managed certificates using a single module if you prefer.
{: .prompt-tip }  