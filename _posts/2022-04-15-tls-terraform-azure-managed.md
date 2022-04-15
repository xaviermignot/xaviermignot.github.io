---
title: "TLS with Terraform and Azure: use managed certificates"
tags: [terraform, azure, app-service]
img_path: /assets/img/azure-terraform-certificates
date: 2022-04-15 08:15:00
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
Note that comparing to the other posts of the series, I have detailed a little bit more what's happening in the code before and after the generation of the certificate. Otherwise I would have only 3 lines of code to show 😬  

> The code snippets above are spread into 3 modules from the repo of the whole series.  
I had previously made another [repo](https://github.com/xaviermignot/app-service-managed-certificate-demo) dedicated to managed certificates using a single module if you prefer.
{: .prompt-tip }  

When browsing the App Service, we can see that the certificate is issued by GeoTrust (aka DigiCert) and of course trusted by the browser:  
![Browser](/04-managed-browser.png) _The certificate is trusted and valid for 6 months ✅_


## Managed certificates: too good to be true ?

As you can see it's very simple to generate a managed certificate, and it's ~~free~~ included in the App Service Plan's pricing. It looks like the perfect solution for securing your Web Apps, but there is a downside that you should be aware of: if you want to use managed certificates, **your App Service have to be publicly exposed**.

So you can't use them if you put your Web Apps behind an appliance like an Application Gateway and enforce the traffic to go through this appliance. 
Even if it's technically feasible to do so (by using tricky maneuvers, I have done this but I won't tell more 😅), this is not a target solution as it will prevent the certificate from being renewed by App Service.

There are other minor limitations like the lack of support of wildcard certificates and the impossibility to export the certificate but it seems fair to me and it did not bother me at all in my personal usage.

> These limitations are listed [here](https://docs.microsoft.com/en-us/azure/app-service/configure-ssl-certificate?tabs=subdomain%2Cportal#create-a-free-managed-certificate) in the documentation, a little bit quietly so I think it's worth mentioning them.
{: .prompt-tip }

To finish on a positive note, these limitations mean that managed certificate might not suit enterprise scenarios where we will probably put a security appliance in front of our App Services.  
But for personal projects, or smaller ones, proof of concepts, or sandbox environments, it's really a great feature, as it makes really easy to expose you applications securely on a custom domain.


### Managed certificates for other Azure services

Staying on a positive track before closing this post, I have to mention that managed certificates are not limited to App Service in Azure.   

I have already used them for Azure [CDN](https://docs.microsoft.com/en-us/azure/cdn/cdn-custom-ssl?tabs=option-1-default-enable-https-with-a-cdn-managed-certificate#tlsssl-certificates) (in GA, supported by the AzureRm Terraform provider) and Azure [API Management](https://docs.microsoft.com/en-us/azure/api-management/configure-custom-domain?tabs=managed#domain-certificate-options) (currently in public preview, no support in Terraform yet).  

It might also exist for other services so if your are about to use an Azure service associated with a custom domain, look for the support of managed certificates 😉


## Wrapping up 

As this post closes the series on certificate generation with Terraform for Azure, let's recap the whole series in a single table:  

| Generation method        | Ease of use | Security level | Summary                                                                                               |
| ------------------------ | ----------- | -------------- | ----------------------------------------------------------------------------------------------------- |
| Self-signed certificates | ✅✅        | 🔓            | You can start here but consider moving to the other method as soon as you can.                         |
| Let's Encrypt            | ✅          | 🔐🔐🔐      | Free & trusted certs, require little maintenance once set-up. <br/> A little harder to understand at first but then can be used everywhere. |
| Managed certificates     | ✅✅✅      | 🔐🔐🔐      | Use this unless your Web Apps can't be directly and publicly accessible. <br/> Or if you are not using Web Apps... |

That's it for this post, I hope this series has been informative, as this is not the topic I am the most comfortable with (don't forget that I am a developer first, and most of us are scared by certificates 😱).  

Anyway, there are several ways to automate certificate generation in Azure and in general, and the industry makes it easier that ever, through open initiatives like Let's Encrypt or public cloud specific features like managed certificates.  

Choose the one that best suits your needs, and thanks for reading 🤓