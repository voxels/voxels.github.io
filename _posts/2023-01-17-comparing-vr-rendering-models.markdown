---
layout: post
title:  "Comparing VR Rendering Models"
date:   2023-01-17 00:00:05 -0000
---

For the past several months I've been working on building environments for the Oculus Quest.  While deploying my environments to device, I learned a few things about what makes lighting in VR look good on device, such as the fact that SM5 shows (much) better results than Vulkan, which means tethering via USB is sometimes advised.  Running on PC is always going to be better than mobile, but how much better SM5 on PC is than Vulkan in device is pretty significant.

<!--break-->

To test the models fidelity, I created two projects with the same assets, one for SM5 and the other for Vulkan.  (This is actually recommended by Epic in the forums).  

I learned that in VR stationary and movable lights are very expensive and you may only get as many as 4, so I changed all my lighting to static (even though static lighting doesn't show up in UE5's Lumen).

Here is an example of the SM5 model running on a cinema camera rail:

{% include youtubePlayer.html id="Wqz57h1Fffo" %}

Here is an capture of the SM5 model running on inside the Quest (excuse the lag, it's introduced by the XBox capture utility):

{% include youtubePlayer.html id="WGjswDTy_nc" %}

And here is the Vulkan model running from an APK on the Quest:

{% include youtubePlayer.html id="oLaJQ3ikrb0" %}

SM5 wins in terms of lighting, and without a capture utility running the frame rate is pretty good in a Quest tethered through USB...  

But there is a use case for packaging into an APK, and that is best done in an entirely separate project with separate lighting controls and maybe even a different set of materials baked for the environment.  I found it easiest to create the SM5 version and then migrate the level to a new Oculus-5.0 branch version of UE5.  

From now on I should always scope a VR project with apk distribution to include a second lighting and material model.

Here are some screenshots from SM5:
