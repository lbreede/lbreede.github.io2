---
layout: post
title:  "🫖 Initial Setup (1)"
categories:
- "VEX Ray Tracer"
tags:
- "houdini"
- "ray-tracing"
- "vex"
---

No more procrastinating. After thinking about all the different ways to put this on the internet, I decided to start a blog, and this is my first blog post.

In August 2019, I uploaded a video on my Vimeo channel, showcasing a ray tracer written in VEX inside SideFX's Houdini.

[![](https://i.vimeocdn.com/filter/overlay?src0=https%3A%2F%2Fi.vimeocdn.com%2Fvideo%2F807849353_1280x720.jpg&src1=https%3A%2F%2Ff.vimeocdn.com%2Fimages_v6%2Fshare%2Fplay_icon_overlay.png)](https://vimeo.com/354673868 "VEX RAY TRACER (+hip)")

Ever since I've been meaning to put up some sort of explanation, video tutorial, class, whatever you want to call it on how to write this thing. Both to teach and to learn how to present ideas and possibly improve on them upon reader interaction.

So here it is, part one of my VEX ray tracer.

***

We start with the initial setup and have to decide on a way how to present the calculated image. I decided to use a projections plane.

![]({{site.baseurl}}/assets/img/vex-ray-tracer/01.001_object_setup.png)

The reason why I'm using a projection plane instead of working straight off of the geometry is the ease of scaling up and down resolution, and therefore performance, by dialing in the grid resolution to your desired size. It also means, that I can stay resolution independent for the objects in my scene.

In this particular example, I have  a camera set to a 4:3 aspect ratio looking straight onto a grid with the same aspect ration (not necessarily the same resolution). I parented to plane to the camera in case we want to do some crazy camera movements down the line.

Furthermore, I got a scene geometry where we can throw in whatever we want to see on the final image. A Utah teapot just seems right.

And lastly an empty geometry container in which the magic happens. Later on we will go back to the scene to create some custom materials and maybe play with the Normals but for now this is how we are gonna start.

***

[Get the scene file here.](https://drive.google.com/file/d/1IlSbHAl71uTOikRcHukCGZEK1rsYbhxz/view?usp=sharing){:target="_blank"}