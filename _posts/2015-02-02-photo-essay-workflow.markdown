---
layout: post
title:  "Automated Photo Essay Workflow"
date:   2015-02-01 14:22:17
categories: wordpress photography
tags:
- wordpress
- python
- photography
- EXIF
- IPTC
- workflow
- LightRoom
---

# Towards a Better Photo Essay Workflow

I've been [blogging][castaway-life] during my extended travels.  This means my usual clunky vacation photo management workflows are getting strained to their limits as I'm constantly going to interesting places to take pictures and write about them.  There is no "after I get back" time to lazily dump work into.  And any time I waste futzing with my computer is time I'm not spending enjoying the marvelous places I'm visiting.

So it behoves me to take some of the technology-focus from my old life and attempt to make this part of my new life flow more smoothly.

## Tools

There's a lot to write about in the workflow — capture with my Sony super zoom and iPhone, management with LightRoom, wrangling photos from one place to the other with Dropbox and cables, selecting the images, writing and publishing the blog posts using MarsEdit. 

But that's not what I want to cover here.  I want to cover a small corner of the process, how to publish quick photo-essays to WordPress.

### The Problem

Here's what I want to do: I want to come back from an event, import all my photos into LightRoom and then publish from LightRoom to my blog as quickly as possible.  There are a bunch of plugins for LightRoom which claim to do the job, but they cost money and/or don't work with LR 4, which is what I use.  

I've got a little scripting skill, so maybe I can scratch my own itch.

My ideal workflow is this: take a series of photos, add little blurbs to each one, write a small introduction and post.  I can easily select a set of photos in LightRoom and export them.  I could then bulk import them into the blog. 

That's where the friction comes in: I'm left with a set of files that I need to either manually add to a post and subsequently view so I can add a description, or I need to modify the metadata in LightRoom to make the subject matter clear from the filename.

Both of these options seem like wasted effort.  What I want to be able to do is to set proper titles and captions in LightRoom and automatically generate a framework post with all the images titled, with long descriptions, and have those descriptions also appear in the body text.  I can then quickly add an introduction and submit the post. The description of the images ought to exist in only one place so I'm not copying it around, and it's the best place for it: my main photo library in LightRoom.

## The Solution

Unfortunately no one has a tool to do this. With nice Python WordPress and image metadata libraries, it should be straightforward to write something to do this, right?

Yes and no.

### Tags and Tags

There are lots of Python libraries which let you read and modify the **EXIF tags**.  [Exifread][exifread] is a single-purpose library for this, [PIL][pil] (or its clone, [pillow][pillow]) has tag functionality built in also.

Trouble is that the 'Title' metadata of JPEG files is not in EXIF.  Caption is, and we can use that for the long description.

Since 'Title' is such a commonplace word and the EXIF tags are the ones folks seem to be after, it took a lot of messing around with Google to discover that JPEG files have another set of metadata in them, the [IPTC][iptc] tags, and that's where the title lives.

### The Script

With that mystery solved, it's a simple matter to write a script to enumerate all the image files in a folder, crack their title and caption, upload the images to WordPress and generate a draft Markdown post using the new image URLs and the caption information.

It's [here][script]

## Final Workflow

1. Capture the images
1. Import them into LightRoom
1. Select the images in LightRoom, do any editing
1. Update the Title and Caption metadata fields
1. Publish the images to a HardDrive folder
1. Run the Script
    1. Enumerate all the images
    1. Rename them based on title: "My Image Title" becomes <folder_name>-my-image-title.jpg
    1. Move them to a processed subfolder
    1. Append the `<img>` tag along with title and description to the Markdown post draft
1. Edit the draft to include an introduction
1. Copyedit (always!)
1. Post


[castaway-life]: http://thecastawaylife.com/blog
[exifread]: https://pypi.python.org/pypi/ExifRead
[pil]: http://www.pythonware.com/products/pil/
[pillow]: https://pillow.readthedocs.org/en/latest/installation.html
[iptc]: http://www.photometadata.org/META-Resources-metadata-types-standards-IPTC-IIM
[script]: https://gist.github.com/iainbryson/b7a86e6cf310034d5c40
