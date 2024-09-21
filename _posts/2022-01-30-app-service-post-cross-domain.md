---
title: "Bug hunting story: Azure App Service, Easy Auth and cross domain POST queries"
tags: [azure, app-service, azure-ad]
media_subpath: /assets/img/easy-auth-cross-domain
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

### Why not building a "quick" demo to demonstrate the issue ?
I have prepared a small repository [here](https://github.com/xaviermignot/azure-easy-auth-cross-domain-post) if you need to see the issue by yourself. It's composed of a Blazor app containing a form element to make POST requests to a .NET 6 API hosted in an Azure App Service.  
As you have probably guessed, the Blazor app is web app A and the .NET 6 API is web app B.  
The following diagram describes quickly the "architecture" of the thing:  
![Diagram](/01-diagram.webp){: width="400"} _Really, an architecture diagram even for such a simple thing ?_  

I have already deleted the Azure resources, so the URLs you will see in the screenshots will not work. If you want to recreate the demo you can follow the instructions in the repo's readme. Basically there is a Bash script that creates the Azure resources using Terraform, builds the code and deploys it (maybe I've pushed things a little bit too far for a "quick" demo...).  

### Well, just a few screenshots will do the trick...
A few screenshots should be enough to understand the issue, starting with the Blazor app:  
![Blazor App](/02-screenshot-app.webp) _The Blazor app containing the form to make a POST request from a domain to another one_
Clicking on the _Send request_ button will submit the form and make a POST request to the API stored in another domain. The first time we do this, we are redirected to _login.microsoftonline.com_ for authentication, and once authenticated to this:  
![Ok request](/03-screenshot-ok.webp) _Once authenticated, we get a 200 response to a GET request_  
Why is there a GET request and not a POST one ? This is because of the identity provider that redirects the user to the web page through a GET request, event if a POST was initially made.  
If we get back to the Blazor app and click on the button again, a POST request is finally made and we reproduce the issue:  
![Error in the browser](/04-screenshot-error.webp) _When the user is authenticated, the POST requests ends up with a 403 response_  
Behind the scene, App Service sets a `AppServiceAuthSession` cookie in the browser, so that there is no need to be redirected to the identity provider each time a request is made.

The following diagrams sums up what just happened in a few steps:
![Workflow](/05-workflow.webp){: width="400"} _The sequence of the actions causing the issue_


## Finding the issue

So how can we track this issue ? I'll try to be brief, but let me share a trick that I've discovered while trying to fix this. In fact your browser's DevTools (I'm using Edge Chromium but I guess Firefox and Chrome do the same thing) can help you to copy the request with its headers for use in another tool.  
Do this go to the _Network_ blade of the DevTools, right-click on the failing request, select _Copy_, and then the "entry" that you want:
![Edge DevTools](/06-screenshot-devtools.webp) _The browser devtools can help to copy the request for replaying it in another tool like Postman or RestClient_  

I personally use the _Copy request headers_ entry to replay the request in VS Code using the great [REST Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client) extension, but you can use curl or PowerShell for instance.
![VS Code REST Client](/07-vscode-restclient.webp) _Replaying the failing request in an external tool can help you to understand what's happening_

What I like about REST Client is the ability to comment some of the headers in the left pane, replay the request and see what changed in the right pane.  
Here is what I've tried in this case:
- Changing the verb from POST to GET results in a 200 response but we already knew that
- Removing the `Cookie` header results in a 302 response with the `Location` header as _login.windows.net_. This is logical as when the `AppServiceAuthSession` cookie is not present, App Service redirects the user to the identity provider, Azure AD in this case.
- Removing or changing the `User-Agent` header while keeping the `Cookie` header results in a 200 response with _Successful POST request !!!_ in the body 🤯. This is quite interesting and explains the _real_ origin of our issue 🤔

So why is the request rejected when the `User-Agent` header says that it was initiated by an internet _browser_ ? Is there some mechanism that prevents the request from being sent from a domain to another one ?  
As you might have guessed the culprit is the Cross-Origin Resource Sharing mechanism, aka _CORS_ ! I thought that CORS was used only for request fired from JavaScript, but in fact it's more complicated that that.  
In our case the combination of cross-domain, POST request, `Cookie` header with session data and `User-Agent` telling that we are a browser "triggers" CORS and therefore blocks the request.


## Fixing the issue

Well, once we know why the issue is occurring it's pretty straightforward to fix it. We just have to update the configuration of our App Service (aka our API) to add the domain of the Blazor app in the list of allowed origins.  
For instance you can do this in the Azure portal:  
![CORS blade in the Azure portal](/08-azure-portal-cors.webp) _The fix in the Azure portal_
But this can also be done using _az cli_, or any IaC tool you use to provision your resources.  
Once it has been applied, wait for a few seconds, the change is not instant even if Azure says it has been updated.  
Just be patient, and then the POST requests will get 200 responses as they were supposed to have !


## Wrapping up

I have to admit I spent way more time on building this demo and writing this post than working on the issue itself. This was also an opportunity to try some cool stuff like Blazor, .NET minimal APIs and of course Bash scripting which is the coolest, right ?  
Anyway it was interesting to learn or re-learn that CORS is not only for the calls made from the Javascript code. I hope this post will help some of my peers, as it might help me in the future.  
If you want to dig more into what is CORS I can't recommend enough the [MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) documentation.  
And overall never step back in front of a bug, even if it seems boring or impossible to solve, keep fixing things you might be about to learn something new.