---
layout: post
title: Manage your Visio shapes with Git
tags: [visio, azure]
image: 
  path: /banner.webp
media_subpath: /assets/img/visio-tips-git-clone
---

In this post I will share a simple trick I have used for many years to keep my favorite Visio shapes always fresh.  
Visio is my go-to tool for architecture diagrams, whether I'm designing something new or documenting something that already exists. I work a lot with Azure, so I like my diagrams with nice, colorful icons of the Azure services I use.  
To my knowledge Microsoft does not provide an up-to-date source for Azure icons, ready to use in Visio.

Fortunately, there are great people on the Internet who provide unofficial but great, up-to-date Visio packages.  
Here are the one I use:
- [Sandro Pereira's Microsoft Integration and Azure Stencil Pack](https://github.com/sandroasp/Microsoft-Integration-and-Azure-Stencils-Pack-for-Visio): the most complete pack, with shapes for all Azure services and much more: devices, frameworks, people, all you need to represent your architecture, the people and the systems around it
- [Azure Kid Azure Stencils](https://github.com/azurekid/Azure-Stencils): less complete than Sandro's pack, but more focus on Azure stuff, so you'll find the icon you're looking for easier with this one. Carefull, the repo is also less active.
- [David Summers' Azure Design](https://github.com/David-Summers/Azure-Design): another great pack, very complete, and the one use the most lately. What I like with this pack is that the connection point are already present on the shapes, so I don't have to add them manually. I'm not a fan of the shadows and the font, so I always change it with my personal settings though

To use this icon sources, I have cloned all the github repos in the `My Shapes` folder on my laptop:
![My Shapes folder](/my-shapes-folder.webp){: width="400"}

So that all the icons are easily accessible in the *Shapes* panel, under the *More Shapes > My Shapes* menu:
![My Shapes menu](/my-shapes-visio-menu.webp){: width="800"}

To keep them up-to-date, I simple go into each folder with my terminal and do a git pull.  
> Keep in mind that Visio must be closed before doing this. 
{: .prompt-warning }
```console
~\Documents\My Shapes\Microsoft-Integration-and-Azure-Stencils-Pack-for-Visio
❯ git pull
remote: Enumerating objects: 9, done.
remote: Counting objects: 100% (9/9), done.
remote: Compressing objects: 100% (7/7), done.
remote: Total 9 (delta 2), reused 6 (delta 2), pack-reused 0
Unpacking objects: 100% (9/9), 3.34 KiB | 27.00 KiB/s, done.
From https://github.com/sandroasp/Microsoft-Integration-and-Azure-Stencils-Pack-for-Visio
   468ca4c..42e806b  master     -> origin/master
Updating 468ca4c..42e806b
Fast-forward
 ...isational Stencils.vssx => Organizational Stencils.vssx} | Bin
 1 file changed, 0 insertions(+), 0 deletions(-)
 rename Others/{Organisational Stencils.vssx => Organizational Stencils.vssx} (100%)
~\Documents\My Shapes\Microsoft-Integration-and-Azure-Stencils-Pack-for-Visio
❯ 
```

And that's it, great resources, a simple git pull, for always fresh Azure icons in my Visio diagrams ! 