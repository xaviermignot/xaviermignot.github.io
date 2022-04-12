---
title: "TLS with Terraform and Azure: use managed certificates"
tags: [terraform, azure, app-service]
img_path: /assets/img/azure-terraform-certificates
---

This post is the last one of my series on the generation of TLS certificates with Terraform for Azure, after the post about [self signed certificates]({% post_url 2022-03-18-tls-terraform-azure-self-signed %}) and the one about [Let's Encrypt]({% post_url 2022-03-28-tls-terraform-azure-lets-encrypt %}).  
For this one we are going to let Azure *manage* everything by using *managed certificates*, a feature available on several services that let Azure handle the generation and the renewal of certificates.


## Previously in the "TLS with Terraform and Azure" series...

If you haven't read my previous post you can check out [this section]({% post_url 2022-03-18-tls-terraform-azure-self-signed %}/#presentation-of-the-context-for-the-whole-series) to get the common context for the whole series.  
> There is also a GitHub [repository](https://github.com/xaviermignot/terraform-certificates) that contains all the code from this series of posts.
{: .prompt-info }


## Level 3: let Azure handle the certificate stuff

The idea behind managed certificates is pretty simple: if you can prove you *own* a domain, then Azure will issue a certificate for you, valid for this domain.  

On the diagram side, things looks way simpler than for the other "levels":  
![Diagram](/04-managed.png) _The diagram looks almost the same as for self-signed certificates_
