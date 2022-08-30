---
layout: post
title:  "Creating a Youtube VR video in Unreal"
date:   2022-08-20 00:00:05 -0000
---

I have been setting up an environment that I can render into Youtube VR from Unreal.  In order to get something that is representative of a high fidelity environment, I first took time to create a landscape, paint it with blended layers, and add assets and textures downloaded from Quixel.

<!--break-->

One preceding exploration included how to use Cesium to create a path over a landscape and animate the camera using a mouse generated heading in Unreal's Sequencer.  I tried exporting this video to 8K at 60fps, but Youtube did not recognize my source material even though I tried converting the file to Youtube's preferred format, webM.  My next step is to try exporting image sequences and using Premier Pro to render the file instead of Unreal.

{% include youtubePlayer.html id="ZvvnyjEen-o" %}

### Generating Landscapes with Unreal

I used Unreal Engine's sequencer to create a camera path through my simulated landscape and move the sun across the background.  The background contains an ocean component, sky sphere blueprint, volumetric clouds, real terrain landscape, landscape component blueprints, blended layer materials, procedural and hand placed foliage.  It is intended to simulate the base layer of where bigger assets would be deposited, and give me an environment I can test camera control and hand tracking in.

{% include youtubePlayer.html id="pVUsu2OgKfw" %}

I lost all this initial work when OneDrive overwrote my documents folder.  Here are the steps I took to rebuild a new landscape:

Importing the heightmap for the landscape
Adding a camera sequence
Adding a VR Hand Tracking pawn
Add landmass custom brush material
Adding an ocean component
Adding a sky sphere blueprint
Adding volumetric clouds
Adding exponential height fog
Adding a post processing volume for auto exposure
Creating a blend material from Quixel textures
Creating procedural foliage from Quixel models

A detailed look at the steps and settings I took can be found in [this post](https://voxels.github.io/rebuilding-a-vr-landscape).

