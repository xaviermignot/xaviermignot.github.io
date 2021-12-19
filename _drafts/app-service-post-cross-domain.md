---
title: "Bug hunting story: Azure App Service, Easy Auth and cross domain POST queries"
tags: [azure, app-service, azure-ad]
---

This is the kind of post that would have saved me some time a few days ago, so let's write it to help others, including future me.  
So I was facing the following situation:
- A web application A is making POST requests to a web application B
- Everything worked fine until I have activated App Service built-in authentication on application B
- Then all POST requests from web app A to web app B ended up with 403 errors 😒
- Disabling authentication on web app B made the problem disappear

A few more info in this:
- Web application B is running in a plain Azure App Service
- Authentication has been added using the built-in authentication of App Service, also called "Easy Auth". Read the doc [here](https://docs.microsoft.com/en-us/azure/app-service/overview-authentication-authorization) if you're not familiar with this feature
- This authentication is used to restrict the audience of web app B without touching its code. It's running on a QA environment whose access is limited to a few Azure AD accounts (of course it's based on Azure AD 😎)
- Web app A and web app B are not running on the same domain (this is some serious hint btw 😏)


## Reproducing the issue

I have prepared a small repository [here](https://github.com/xaviermignot/azure-easy-auth-cross-domain-post) if you need to see the issue by yourself. It's composed of a Blazor app containing a form element to make POST requests to a .NET 6 API hosted in an Azure App Service.  
As you have probably guessed, the Blazor app is web app A and the .NET 6 API is web app B.