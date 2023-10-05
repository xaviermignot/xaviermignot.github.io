---
title: "How to run a Minecraft service in an Azure container in 2023"
tags: [azure, bicep, azure-container-instances, azure-container-apps]
img_path: /assets/img/minecraft-azure-container
---

Recently I have found myself (again) in a Minecraft phase, this happens regularly. This time I wanted to run a server for my family, obviously I have chosen to run it in Azure. I will share in this post what I have learnt on the way.

## TL;DR The inspiration post 
I had read this [post](https://markheath.net/post/minecraft-server-azure-container-instances) from Mark Heath a few years ago, and reminded it before setting my server.  
Basically I wanted to start from there and use the latest tools and Azure services. Before going further, if you just want to run you server in the most simple way, the solution described by Mark and completed by the comments on his post is still on point, so I can't recommend enough to check it out.  
That said, you can keep following along if you want to know how to slightly modernize it.

## Choosing the right service for the job
At first I didn't want to use Azure Container Instances, and have considered Azure Container Apps, at that time the new kid in town. Well there were some limitations on the way, depending on the Minecraft edition. Let me summarize everything in this table:

| Minecraft Edition                          | Server protocol   | Compatible Azure Services                            | What you need to know                       |
| ------------------------------------------ | ----------------- | ---------------------------------------------------- | ------------------------------------------- |
| Java Edition<br/> _The original_           | TCP on port 25565 | Azure Container Instances <br/> Azure Container Apps | Runs on Windows, Linux & macOS |
| Bedrock Edition<br/> _The rebuilt edition_ | UDP on port 19132 | Azure Container Instances only                       | Runs on Windows 10 & 11, consoles and mobile devices <br/> Not compliant with Container Apps due to lack of UDP support  |

As you can see, as Azure Container Apps don't support UDP, so we can't use them for the Minecraft _Bedrock_ edition.  
Personally, I enjoy playing the Bedrock edition. I know it's an unpopular opinion in the Minecraft community, but it runs smoother on my laptop, and playing together locally is way easier with it, so it's my family's edition-of-choice.  
So I have chosen the good old Azure Container Instances for my server. I have tried to run the Java edition in Azure Container Apps, and I confirm it works.

### Anyway, was Container Apps made for this ?
Initially I was considering Azure Container Apps as the ultimate replacement for Container Instances. It turns out I was wrong, I guess now that both services will continue to exist.  
Moreover, running a Minecraft server in Azure Container Apps is not a good usage of the service. This service suits applications with scaling needs, which is not the case of a Minecraft server. A Minecraft server simply doesn't scale, as all players are playing together on the same instance (this is not a massive multiplayer online game where the players are spread across several instances).  
So finally, using the latest service is not always the best idea, especially when existing services do the job in a simpler way.

## Adding some IaC flavor