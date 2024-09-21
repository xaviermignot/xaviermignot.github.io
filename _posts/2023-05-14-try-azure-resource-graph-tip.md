---
title: "Quick tip: Try Azure Resource Graph"
tags: [azure, quick-tip, azure-monitor]
media_subpath: /assets/img/try-azure-resource-graph-tip
image:
  path: banner.webp
date: 2023-05-14 18:30:00
---

This a quick post to share a service I have discovered this week: Azure Resource _Graph_ Explorer.  
Apparently it has been around since a couple of years, but I have only found out about its existence this week. Let's see how awesome this service is.

## What is Azure Resource Graph ?
Basically it's a service (living [here](https://portal.azure.com/#view/HubsExtension/ArgQueryBlade) in the portal) that let you query your Azure resources using KQL.  
Until now when I needed to dig into my resources I used Azure Resource Explorer (from the [dedicated site](https://resources.azure.com) or from the [portal](https://portal.azure.com/#view/HubsExtension/ArmExplorerBlade)), but I have never found it easy to use and was always wondering if I should start searching by _providers_ or by _subscriptions_.  
Using Azure Resource _Graph_ is much easier and powerful, if you want to know more about how to get started with it I recommend checkout the [documentation](https://learn.microsoft.com/en-us/azure/governance/resource-graph/).

## What can we do with it
For now I have barely scratched the surface of the service but I can already share a couple of use-cases.

### Query resources across subscriptions
The first thing you can do right away is query your Azure resources, and the power of the tool is that you can do this across _all_ the subscriptions you have access to. This is extremely useful in an enterprise environment with a bunch of subscriptions for your various landing zones (with the CLI or PowerShell, you have to change your context to achieve this).  
With Azure Resource Graph, a simple KQL request on the `resources` table will give you everything, and then you can refine the results by type, and do things like getting the number of VMs by size, location, etc.  

> This [blog post](https://techcommunity.microsoft.com/t5/itops-talk-blog/azure-resource-graph-zero-to-hero/ba-p/2303572) is a great read to find out how you can query your resources using Graph Explorer.  
{: .prompt-info }

> Also keep in mind that using it from the portal is just a start, then you can start thinking about using it in your scripts using [Azure CLI](https://learn.microsoft.com/en-us/azure/governance/resource-graph/first-query-azurecli), [PowerShell](https://learn.microsoft.com/en-us/azure/governance/resource-graph/first-query-powershell) or [REST](https://learn.microsoft.com/en-us/azure/governance/resource-graph/first-query-rest-api).
{: .prompt-info }

### Get the payload of an Azure Monitor alert
Working with Azure Monitor, you often need to check the payload of an alert for troubleshooting reasons. Until now I was doing it with an action group and a webhook to a service like [Webhook.site](https://webhook.site/). But there is a better way.  
From the Azure Monitor Alerts blade in the portal, select the occurrence of your alert and copy its _id_:
![azure-monitor-get-alert-id](/01-get-alert-id.webp)_I have cropped the screenshot and masked some details_

Then go to the Azure Resource Graph Explorer blade, and copy this query with your alert id:
```kusto
alertsmanagementresources
| where ['id'] == '<YOUR ALERT ID HERE>'
```
Click on _Run query_ (or use the `shift+enter` shortcut like a boss 😎), and click on the _See details_ link at the very end of the single result row:
![query-results](/02-execute-query.webp)

Go to the _properties_ field to grab the full payload of your alert:
![get-payload](/03-get-results.webp)

Using this payload you can try to figure out why your processing rule is not catching your alert, or trigger manually your Logic App or Azure Function without waiting for your alert to occur.

## Wrapping up
That's it for this post, I'll try to do "quick tips" like in the future if I manage to write post... quickly.  
In the meantime, go check out this service if you haven't already 🤓