
The problem of lighting 3D scenes in hardware accelerated rasterized rendering is really hard to solve. There are no 100% solutions, and what seems perfectly obvious to the uninitiated (light just bounces around) is extremely hard to do in practise. The following blog post is about one such method that I wanted to try out for a long time.

Code
The code for this demo is on Github


Screenies

Contents
![image info](/screens/screenie1_thumb.jpg)
![image info](/screens/screenie2_thumb.jpg)
![image info](/screens/screenie3_thumb.jpg)
![image info](/screens/screenie4_thumb.jpg)



## Model
I needed a test model for this idea. For other rendering experiments you usually use things like the stanford bunny and such. But for global illumination you need something a bit more architecture oriented.

There is a very nice model by Marko Dabrovic, the Sponza Atrium. I've had a bit of trouble with those files, but Morgan McGuire has a superb version of it that worked very nice.

## Overview
The basic idea of the technique is to:

Sample the scene from many light probes
Calculate the diffuse spherical harmonics for the probes
Apply this lighting back to the scene
Repeat a few times for each bounce
The Demo also implements a variety of other techniques such as:

FXAA Anti-aliasing
Variance shadow mapping
Variance screen space ambient occlusion
Bumpmapping
Screen space depth displacement
Gamma corrected rendering
Debug viewer
Deferred shading
I'll briefly explain each of the steps now and how it was made to run in realtime.

## Sampling the Scene
The original scene has nearly 200'000 triangles and it's not light mapped. Lightmapping in realtime rendering is to store a set of UV texture coordinates for the scene such that no two surfaces overlap, and a unique texel is used to texture every part of the scene.

I created a simplified scene model in Blender that was properly light mapped.

![image info](/screens/simplified-model.jpg)


![image info](/screens/lightmapped.jpg)


## Lightprobes
In order to sample the scene I need to render it from the point of view of 47 lightprobes. These probes are 6-side cubemaps in a low resolution. A naive approach would be to render the scene 47 times, but that would be too slow. So instead I build a dictionary that maps to UV coordinates in the lightmap as well as the diffuse color of the scene from the probe.

![image info](/screens/probes.jpg)

The dictionaries are built actually rendering the scene 47x. But for the live version of the demo they're just loaded from images since they are static long as the probe positions don't change.

Then when I want to compute each probe, I can just use the dictionaries to update them:

![image info](/screens/sampling.jpg)


## Irradiance
The cubemaps need to be converted to something expressing diffuse irradiance. I have toyed with irradiance in two previous demos. This time I use spherical harmonics.

Spherical Harmonics
It sounds awfully complex, but it isn't. There's a few blog entries on WebGL and spherical harmonics. The wikipedia entry will highlight some of the concepts.

The Orange Book (OpenGL Shading Language) has a bit of explanation/code about it in chapter 12.3 .

There are also very nice sources on Spherical harmonics on the web such as:

An Efficient Representation for Irradiance Environment Maps from Stanford Universities Computer Graphics Laboratory by Ravi Ramamoorthi and Pat Hanrahan
Spherical Harmonic Lighting: The Gritty Details by Robin Green
Spherical Harmonics by Volker Schönefeld
Stupid Spherical Harmonics Tricks by Peter-Pike Sloan from Microsoft

### Summing Up
I need to simply sum up the coefficients for each of my 47 lightprobes from my 47 little cube maps. Graphically that comes out like this:

![image info](/screens/harmonics.jpg)


So I get my 47 light probes for the scene

![image info](/screens/screenie5_thumb.jpg)


## Direct Illumination
I am using a directional light (as in the sun) to introduce light to the scene. In order to do that I need shadow mapping.

Shadow Mapping
The technique I use is called variance shadow mapping (excelent introduction in this GPU Gems 3 article and on the page of the author). In order to use it, first we need to render the scene from the lights point of view and store the scaled to 0-1 depth and depth squared and blur that a bit.

![image info](/screens/sundepth.jpg)

Then we can use that to either shadow map the scene

![image info](/screens/screenie6_thumb.jpg)


Or apply it to the lightmap by rendering the simplified mesh using the texture coordinates for gl_Position.

![image info](/screens/shadowmap.jpg)

And as we've seen in the Irradiance section, once it is in the lightmap, I can use it to sum up the lightprobes coefficients.

## Indirect Illumination
Once I have the direct illumination summed to my lightprobes coefficients, I can use this to apply it to the lightmap. I simply render sufficient triangles for each lightprobe into the lightmap space looking up my coefficient texture and computing the spherical harmonics function. A few notes on how that works.

The indirect illumination lightmap is set to additive blending and is a floating point texture. I add the computed light contribution from spherical harmonics to rgb and 1 to alpha. When I want to use this indirect illumination lightmap, I divide rgb/alpha so as to get an average of all overlapping lightprobes.

![image info](/screens/screenie6b_thumb.jpg)


I can then restart the process from the beginning and compute a new set of spherical harmonics coefficients, in this way, gaining a second, third and so on bounce.

## Preparing to render the Scene
So far all of what I explained only applies to any change of light. We have not yet rendered the scene with more polygons from the observers point of view. Since we use deferred shading, We first need to render the scene two times capturing two characteristics.

Albedo
Albedo is the diffuse, unshaded color of each pixel on screen. This is done by simply outputting the appropriate textured color for the scene view.
![image info](/screens/screenie7_thumb.jpg)


Normaldepth
In order to do lighting calculations we also need the normal and position of each pixel on screen. We could capture this in two passes (writing normal to rgb and in a second writing position to rgb). But this is wasteful and I do it in one pass by storing normal to rgb and depth to alpha.

![image info](/screens/screenie8_thumb.jpg)


Normalmapping
The normaldepth output also contains normalmapped normals and displaced depth. In order to do this I deliver bumpmaps (height) to the normaldepth shader. And I use a technique described by Morten S. Mikkelsen from Naughty Dog Inc. called "Bump Mapping Unparametrized Surfaces on the GPU".

Ordinarilly you would need to deliver a tangent space to do normal mapping. But it is possible to do normal mapping using standard derivatives (which are good for other things as well).

## Applying Irradiance with Deferred Shading
This renderpass samples the normaldepth and renders a coarse sphere for each probe. The depth buffer of the previous render is re-used for this pass and the depth test is set to depth test GEQUAL and depth writing is disabled. This technique is commonly known as deferred shading and it discussed all over the internet as well as in the ShaderX and GPU Pro series of books.

Again blending is set to additive and the output is a floating point texture. So when looking up into this render, again we divide rgb/alpha. The output of this pass looks like this:

![image info](/screens/screenie9_thumb.jpg)

## Variance SSAO
Earlier I've talked about variance shadow mapping. In ShaderX7 Angelo Pesce talks about using variance calculations to do SSAO in the article "Variance Methods for Screen-Space Ambient Occlusion" that would be similar to the blur and unsharpen mask method used in early Crytek engines. There wasn't a detailed explanation how that works, so I've just tried around a bit and arrived at something usable.

![image info](/screens/screenie10_thumb.jpg)


Gamma Correct Rendering
When experimenting with fairly subtle lighting effects and HDR images, it is important to factor out gamma from your rendering. There is an excelent GPU Gem on the importance of being linear that explains this extremely well.

## FXAA
The fabulous Timothy Lottes shares his implementation of anti-aliasing with the public. For this demo I used FXAA 3.11 and ported it to WebGL.

## Misc.
Debugging

![image info](/screens/screenie11_thumb.jpg)

I would recommend anybody doing graphics development to come up with such a tool. It is extremely helpful.

Spherical Harmonics
Usually there's a set of 5 constants used in conjunction with spherical harmonics. There are entire research papers on how to calculate these constants. Every source I consulted on spherical harmonics used a different set of constants. The Orange Book coefficients wouldn't work with different constants than the one in the orange book etc.

This led me to experiment with the constants and I found that there is absolutely no right way to calculate them. They're pure artistry. This is why I included a section in the GUI to modify the constants rather then setting them to some defined values fixedly. If you use spherical harmonics for lighting, you will need to tune your constants to look nice for the flavor of lighting you want to achieve.

## Libraries
Other bits of useful things used in this demo are

dat.gui, a superb way to manage lots of little variables troughout your program.
JQuery, a very nice abstraction layer over DOM handling
stat.js, a very useful FPS/rendertime tracking utility, thanks MrDoob
Thanks
Big thanks also goes out to my testers and the people who endured me during this time including:

jareiko
jdash
Nightmare
timb
Cheery
From freenode.net on #webgl

## Future Work
The deferred shading part uses most GPU resources. Thorough testing revealed this to be due to the massive amount of overdraw. If this is to sped up I need to address the overdraw issue.

An interesting way to do that would be to analyze the existing depth and come up with a tiled rendering way to reduce overdraw. Some interesting discussions on these ideas can be found below:

Gamerendering on Light Indexed Deferred Rendering
Tiled Shading presented at HPG 2011 by Ulf Assarsson
Clustered Deferred and Forward Shading presented at HGP 2012 by Ola Olsson, Markus Billeter and Ulf Assarsson
Deferred Rendering for Current and Future Rendering Pipelines by Andrew Lauritzen from Intel presented at Siggraph courses 2010

