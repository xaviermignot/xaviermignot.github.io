---
title: "Bug hunting story: Azure App Service, Easy Auth and cross domain POST queries"
tags: [azure, app-service, azure-ad]
img_dir: /assets/img/easy-auth-cross-domain
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
The following diagram describes quickly the "architecture" of the thing:  
![Diagram]({{ page.img_dir }}/01-diagram.png){: width="400"}  
And here is another diagram to show the sequence of the actions:  
![Workflow]({{ page.img_dir }}/02-workflow.png){: width="400"}  
If you want to recreate my demo in you own subscription you can follow the instructions in the readme of the repository. But if you are here you might be facing the issue already so let's put a few screenshots before moving on to the solution.  


## Fixing the issue

## Wrapping up

I have to admit I spent way more time on building this demo and writing this post than working on the issue itself. This was also an opportunity to try some cool stuff like Blazor, .NET minimal APIs and Bash scripting which is the coolest, right ?  
Anyway it was interesting to learn or re-learn that CORS is not only for the calls made from the Javascript. I hope this post will help some of my peers, it might help me in the future.  
And never step back in front of a bug, event if it seems impossible to solve, keep fixing things you might be about to learn something new.