---
layout: post
# title:  "Starcrossed Outline Breakdown"
date:   2021-05-17 19:07:06 -0700
categories: jekyll update
---
# Introduction
One of the defining features of Starcrossed's visuals is it's use of sillutes. Each visual element in Staracrossed was
designed to have a unique recognizeable shape. We wanted different gameplay elements (players, enemies, items, etc.)
to be distiguishable by there shape on screen to prevent confusion in a fast passed environment. Through development it 
became apparent that in order to have colorful backdrops and keep this design criteria, we would need something to make 
foreground elements pop against the background. So we created a custom outlining effect and rendering pipeline to allow
us to do this. This post is a dive into how that effect works, as well as the optomization strategies we used to keep
the effect running at 60 fps even on lower end PCs and consoles.

# Finding an Outlining Technique
Outlines aren't anything new to games. At this point there so common in Unity games that many Unity shader tools come with 
a built in way to add an outline to a shader. The most common way of accomplishing this effect is to use a an extra draw 
pass of your outlined object. The first draw pass draws the outline, by drawing an inflated version of the mesh that is 
being outlined. This can be done by just moving the vertex out along its normal with a vertex shader. If you draw this 
inflated version with no depth, then only color will be written. You can then draw the normal version of the mesh using the 
normal shader. It will draw inside the inflated version, can create an outlined effect. Here I've done this approach in 
Starcrossed.

There is however a problem with this approach. As you may notice, then lines are somewhat inconsistent with there 
thickness. Depending on the mesh we use we will get very different consistencies of the line thickness.

On smooth surfaces we generally get a thicker line than on rough surfaces. If a mesh is composed of both, then we will get
varying thickness on different parts of the mesh. So why is this? We can answer this using some basic math. When we think of
outline thickness, we generally think of the distance we will draw the outline from the surface. We can think of this
distance in world space of screen space, it doesn't really matter. With our vertex shader we do approximetly this math.
We move a vertex V, along normal N a distance D, which should give us a new vertex V' that is distance D from V. V' should
be D away from the surface. However this is only true at the vertex points themselves. Lets take a points beween two
verticies V1 & V2 called P. We can think of all properties of P as being a linear interpolation between V1 & V2. This
should mean its normal should be Np = Lerp(Nv1, Nv2, 0.5). The outline should thus be distance D along this vector away
from the surface.

Now if we look and measure the actual distance at that point, we can see its a value less than D. Using some trig, we can
see why pretty easily. Since a vertex shader can only move at a resolution of the verticies. We can only actually move
points V1 and V2. P will be moved in relationship to these points. If V1 moves a distance D away, P will move a 
D*cos(angle) away, or put another way, P will move like the leg of a triangle. The closer the normal is to the normal of P,
the less noticable this is, which is since cos(angle) as angle approaches 0 approaches 1.

With the importance of keeping clean siloutes, we needed a way of creating a uniform outline, that would be an equal distance from the surface at all points. For insperation we looked at a few other games that accomplished some
simialr effects, one of the main ones being the Borderlands series. These games all used a more complex form of the
outline effect by doing it as a post-processing step. Depending on the style they all had there own quarks to it, but
the basics were the same. 

By using an edge detection algorithim we can find edges within a render buffer. We can then apply this edge buffer with other operations to draw the outlines (and outline colors) where we want. Since this operation is done in screen space, the width of an edge can be determine at a pixel resolution, we can get a uniform line at all points.

# Starcrossed Outline Pipeline
Although we had a technique, it turns out there are many ways of implementing this effect, It could be done as entirely a 
post-processor, or a pre-processor, or a mixture of the two even! With starcrossed we had to make a pipeline that would 
allow us to just outline the outer edge of objects, and allow us to chose a color based on the object we were outlining.

Our implementation changed greatly over the course of development, but the concepts of how it worked remained mostly the
same. 

To start we have unity draw a render target that is the same size as the screen, but only has a 8bit red channel. For all 
objects that need an outline we render them to this buffer, using a simple shader that only outputs red. The result is a 
buffer that shows every pixel where an outlined object lives. We refer to this buffer as the object buffer.

From this buffer we run an edge detection algorithim on the object buffer. For us we do this through a simple pixel shader
that samples points around the current pixel in the object buffer. If any of those pixels are higher than a given value
then we write into our pixel, otherwise not. To change the thickness of the line we can simply change the distance at which
we do this sample. Sampling farther away gives us a thicker line. 

We can also change how accurate the line is but increasing the number of samples. For example four samples (one in each
cardinal direction) will give a blocky result.

Using higher numbers like 8 or 16 gives a smoother look, at the cost of performance. We'll dive more into that later!

We now have a buffer that tells us where in screen space we want to draw outlines! However this buffer is just a solid
color, and we needed to have many different colors depending on what we're outlining. There are a lot of different
ways to tackle this problem. Our solution was to use what we called the AOE (or Area Of Effect). For every object
we render with an outline, we create a mesh that is invisible to the normal camera. This mesh is usually just a circle
with the outline material.

To get the colored outlines, we simply only draw the AOE meshes where in screen space there is an outline in the outline
buffer. The end result is a nice colorful outline!

# Optimizing
Our early iterations of the outline effect were...not very efficent to say the least. Most low end devices struggled to
render at 60 fps, and on very low end devices, like the switch, we'd struggle to even get 20 fps. However the effect
itself, even in early versions was not very intensive. It used very little CPU and GPU compute power to be able to
render. There were some compute optimizations that I'll touch on at the end, but the major changes that were made
all revolved around one culprit, stalling!

In moder compute devices, the hardware can generally handle more than one task at time. On CPUs we know these as cores, but GPUs also have similar setups. Stalling occurs where not every compute unit is given a job, generally due to waiting on something. For example if your program was waiting on reading a file, this would be a stall, since no compute unit has any work while this is happening. For a games rendering, any stall is really bad, we want to make sure that at an time both GPU and CPU are being given work for all compute units, or as many is possible at the very least.

For us, our orginal implementation of the pipline was very stall prone. In code it looks very simple:

```
objectBufferCamera.drawToRenderTarget(objectBuffer, objectShader)
Graphics.Blit(objectBuffer, outlineBuffer, outlineShader)
aoeCamera.drawToRenderTaraget(aoeBuffer, aoeMask)
Graphics.Shader(blendShader, sceneBuffer, outlineBuffer, aoeBuffer)
```

It required two extra camera draws, one for the object buffer, and one for the aoe buffer. And then two extra
graphics calls, one to get the outline, and one to do the final blending.

When we profiled this we saw a lot of CPU time being taken on these individual commands, however not a lot of CPU procesing.
We saw similar behaviour on the GPU as well. This implied to us that the CPU and GPU were being given a small load of work
but in such a way that they couldn't work on anything else. This creted a bottleneck in the pipeline at the point where
we rendered our outline.

One of the problems we encountered was just with how Unity handles its old drawing APIs. These APIs gurantee certain side
effects when they finish. So even though many of them could be async, they aren't. This meant things like our 
`Camera.Render` calls were blocking the CPU while the GPU did the work. Since this was happening at the end of the frame
there was no CPU work to handle while waiting, and no additional GPU work to handle while waiting so we were limiting the
workt he CPU could do to zero and the GPU to our one task at a time.

Our first change was to move this task to `CommandBuffers`. CommandBuffers allow you to record GPU commands ahead of time
in a buffer. You can then either submit them all at once, or attach them to the Unity rendering pipelien so they just
happen as a part of the frame rendering. This ends up helping stalling two fold. Our CPU doesn't have to wait on the
buffer if its just a part of the frames commands, so the CPU can do other game logic now after creating the commands.

The GPU also gets quite a speed up. Modern GPUs have a concept of queue pipelining. Pipelining is the ability to take
a set of sequential commands, and parallise them based on dependencies. We can see this easily with a piece of code like:

```
int a = 0;
int b = 2;
int c = a + b;
int d = 4;
int z = 3;
int y = z + d;