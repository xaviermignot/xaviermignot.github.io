---
layout: post
title: "Microsoft Visio 101: The basic tools"
tags: [visio]
img_dir: /assets/img/visio-tips-tools
image: 
  src: /assets/img/visio-tips-tools/banner.png
---

A I really enjoy making architecture diagrams using Visio, I want to share in a few posts some tips to help you do the same.  
This one is about the basic tools, Visio is a complex software so it's important to know basic stuff and not feel overwhelmed.


## Meet the Visio "tools"

To get started you should get comfortable with the *tools* of Visio, which are:
- The pointer tool, to select and drag shapes
- The text block tool, to move the text associated to shapes
- The connector tool, to draw lines between shapes
- The connection point tool, to edit the connection points of a shape  

All the tools are located in the `Home` tab of the ribbon, in the `Tools` toolbar:  
![Tools toolbar]({{ page.img_dir }}/01-tools-toolbar.png)  
It's important to know their respective roles, and how to switch between them, it's quite simple you will see 🤗  

### The pointer tool
The pointer tool is the most basic, selected by default. You can use it to select, drag, copy shapes for instance.  
Let's start by creating a rectangle in a blank diagram by clicking on the white rectangle in the toolbar:  
![Create rectangle]({{ page.img_dir }}/02-draw-rectangle.png)
_(This is another tool I didn't mentioned, you can use to draw basic shapes it's pretty straightforward 😉)_  
Now you're in the *rectangle creation mode*, so any click & drag in the diagram zone will draw a rectangle. If you want to stop drawing them and start moving them for instance, select the pointer tool again using the `Ctrl+1` shortcut (or clicking on the pointer in the toolbar but memorizing the shortcut is worth it).  
Then you'll be able to move around those rectangles, resize them, delete, any basic action you can expect from a diagram drawing tool.  
That's all the pointer tool is about: getting back to the most basic mode of the tool, using the shortcut to memorize: `Ctrl+1`  
Let's keep a single rectangle before moving to the second tool.

### The text block tool
In Visio you can easily add some text to any shape: simply select the shape, start typing and that's it. You can hit F2 before typing but it's not mandatory.  
So there is no need to manually associated a text zone to a shape, it's already within each shape.  
You can use the text block tool to move or resize these text zones, without changing the shape itself. Select a shape, hit `Ctrl+Shift+4` and arrange the text as you want.  
![Text with shape]({{ page.img_dir }}/03-text.png)  

### The connector tool
Connecting shapes is where Visio begins to shine in my opinion. The first tool for managing connection is the connector tool which creates connections between shapes.  
To use it we are going to use more complex shapes than rectangles. Let's select the first shape available in the *Shapes* panel, the *Main topic* shape in the *Brainstorming Shapes* stencil, and create two shapes:  
![Main topic shape]({{ page.img_dir }}/04-main-topic.png)  
These shapes have predefined *connection points*, so if you hit `Ctrl+3` to switch to the connector tool, you can easily draw a connection between the two shapes.  
![Connect the left shape]({{ page.img_dir }}/05-connection1.png)
_Draw a connection from one point in the green square..._  
![To the right shape]({{ page.img_dir }}/06-connection2.png)
_...to another connection point_  
As the connection points anchors the connection to the shapes, you can move the shape around and the connection will follow naturally.

### The connection point tool
As all shapes do not come with predefined connection points, or sometime you need to add some, that's where the connection point comes in handy. 
Select a shape using the pointer tool, hit `Ctrl+Shift+1` to switch to the connection point tool, and you can perform tasks like:
- Adding a new connection point using `Ctrl+click`
- Dragging an existing connection point
- Delete a connection point by clicking on it and hit `Delete`  
The connection point tool combined with the connector tool let you do anything you want with connection between shapes.


## Wrapping up

That's it for this post, we have covered the most basic features of Visio. Next time we will see how to use the align and position features to perfectly place shapes in a diagram.  
Meanwhile, take some time to memorize the keyboard shortcuts and the role of each tool:  

| Tool | Shortcut | Role |
|------|----------|------|
| Pointer tool | `Ctrl+1` | Select, move & resize shapes |
| Text block tool | `Ctrl+Shift+4` | Arrange text linked to shapes |
| Connector tool | `Ctrl+3` | Draw connections between shapes |
| Connection point tool | `Ctrl+Shift+1` | Edit shapes connection points |