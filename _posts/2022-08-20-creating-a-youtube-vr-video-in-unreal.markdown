---
layout: post
title:  "Creating a Youtube VR video in Unreal"
date:   2022-08-20 00:00:05 -0000
---

I have been setting up an environment that I can render into Youtube VR from Unreal.  In order to get something that is representative of a high fidelity environment, I first took time to create a landscape, paint it with blended layers, and add assets and textures downloaded from Quixel.

<!--break-->

One preceding exploration included how to use Cesium to create a path over a landscape and animate the camera using Unreal's Sequencer.  I tried exporting this video to 8K at 60fps, but Youtube did not recognize my source material even though I tried converting the file to Youtube's preferred format, webM.  My next step is to try exporting image sequences and using Premier Pro to render the file instead of Unreal.

{% include youtubePlayer.html id="ZvvnyjEen-o" %}

### Generating Landscapes with Unreal

I used Unreal Engine's sequencer to create a camera path through my simulated landscape and move the sun across the background.  The background contains an ocean component, sky sphere blueprint, volumetric clouds, real terrain landscape, landscape component blueprints, blended layer materials, procedural and hand placed foliage.  It is intended to simulate the base layer of where bigger assets would be deposited, and give me an environment I can test camera control and hand tracking in.

{% include youtubePlayer.html id="pVUsu2OgKfw" %}

After generating the landscape and camera sequence, the next step was to use the Panorama Camera Plugin to generate the image sequence for a stereoscopic view of the camera, and take the results into Premier Pro to render the video.

I've also been working on transferring the VR pawn from the VRTemplate sample project into my own projects.  I'm starting off by rendering the basic third person shooter sample project in VR to find out what is contained in the template's blueprints, and then I want to drop the VR pawn into my virtual landscape to see if the Oculus can render the Quixel assets (in the sample projects it seems like they maybe can't.)

I learned along the way that the docs for Oculus are wrong..  you need to use VS 2019 instead of 2017, and I also learned that you can't run the Oculus repo from an external USB drive or copy it from one place to another.  The only way it builds and runs is to clone the repo to the internal drive and build with VS 2019.  
