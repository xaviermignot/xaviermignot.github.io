---
title: Using WebP images for this blog
description: How I have converted all the images of this blog to WebP instead of PNG or JPEG.
date: 2024-12-27 23:30:00
---

I have recently seen many posts about websites good practices measured by tools like [Lighthouse](https://developer.chrome.com/docs/lighthouse/) in Chrome-based browsers. I regularly audit this blog and I have improved its score by converting all the images into WebP.  
Let's see quickly how you can do this for your website as well.

## What is WebP and how to get started 
[WebP](https://developers.google.com/speed/webp) is an image format developed by Google, now open source and [supported](https://caniuse.com/webp) by all major browsers. Using WebP instead of PNG/JPEG promises better compression, performance and SEO (check the links for real information about this 🤗).  

Google ships utilities to convert images from and to WebP, you'll find installation instructions [here](https://developers.google.com/speed/webp/docs/precompiled). As a lazy Linux user (WSL2 with Windows 11), I have used Homebrew with the command `brew install webp`.  

The utility we are going to use is `cwebp`, to convert images from PNG/JPEG to WebP.

## What to do once the tools are installed ?
Alright, so using this `cwebp` tool is not rocket science, as shown in the [docs](https://developers.google.com/speed/webp/docs/using#using_cwebp_to_convert_images_to_the_webp_format) it's pretty easy to convert a single image.  
But this blog has been running for a few years now so in my case I had more than hundred images spread across dozens of folders to convert. How to achieve that ?  

Well, it requires some Bash knowledge I didn't had at the time. Enough talking, let me spare you some googling time and share the one-liner I came up with:
```shell
for i in **/*.png; do cwebp "$i" -o "${i/png/webp}"; rm "$i"; done
```
Running this from the assets folder of my blog converts all the PNG images to WebP, keeps the same folder and file name as the originals, uses the `.webp` file extension, and removes the originals.  
Replace `png` by `jpeg` to do the same thing for JPEG files:
```shell
for i in **/*.jpeg; do cwebp "$i" -o "${i/jpeg/webp}"; rm "$i"; done
```
Then a few _search and replace_ across the markdown files and the job is done !

## Keeping on converting everyday
Now that the historical images have been converted, to convert new images easily I needed an _alias_ instead of searching the above command in my history.  
For a reason I don't remember, using the one-liner directly as an alias did not work, so I ended up putting this in my `~/.zshrc` file:
```shell
function png2webp() {
  for i in **/*.png; do
    cwebp "$i" -o "${i/png/webp}"
    rm "$i"
  done
}

alias png2webp='png2webp'
```
{: file=".zshrc" }

And I use the `png2webp` alias to convert images before publishing a new post.

## Wrapping-up
That's it for this post, a quick article to share some commands to follow the trend of image format for your website. Regarding my _score_ in Lighthouse, it was slightly better after the conversion to WebP, but not that much honestly (probably because my PNGs were already optimized using [tinify](https://tinypng.com/)).  

There are still room for improvement on this website, I will continue to improve my score and encourage you to do so 🤓 
