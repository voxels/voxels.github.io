---
layout: post
title:  "Creating a Youtube VR video in Unreal"
date:   2022-09-13 00:00:05 -0000
---

I have been setting up an environment that I can render into Youtube VR from Unreal.  In order to get something that is representative of a high fidelity environment, I first took time to create a landscape, paint it with blended layers, and add assets and textures downloaded from Quixel.

### Tldr:

I made this landscape for the Oculus and Youtube in 4k and VR:

{% include youtubePlayer.html id="PoAqnRDY94U" %}

<!--break-->

One preceding exploration included how to use Cesium to create a path over a landscape and animate the camera using a mouse generated heading in Unreal's Sequencer.  I tried exporting this video to 8K at 60fps, but Youtube did not recognize my source material even though I tried converting the file to Youtube's preferred format, webM.  My next step is to try exporting image sequences and using Premier Pro to render the file instead of Unreal.

{% include youtubePlayer.html id="ZvvnyjEen-o" %}

### Generating Landscapes with Unreal

I used Unreal Engine's sequencer to create a camera path through my simulated landscape and move the sun across the background.  The background contains an ocean component, sky sphere blueprint, volumetric clouds, real terrain landscape, landscape component blueprints, blended layer materials, procedural and hand placed foliage.  It is intended to simulate the base layer of where bigger assets would be deposited, and give me an environment I can test camera control and hand tracking in.

{% include youtubePlayer.html id="pVUsu2OgKfw" %}

I lost all this initial work when OneDrive overwrote my documents folder.  Here are the steps I took to rebuild a new landscape:

- Importing the heightmap for the landscape
- Adding a VR Hand Tracking pawn
- Compiling for the Oculus
- Add hand tracking to the pawn
- Add teleportation to the controllers
- Add a VR movement sequence
- Set up the landscape with Quixel materials
- Adding blueprint brushes to the landscape
- Painting the landscape
- Adding an ocean component
- Adding a sky sphere blueprint
- Adding volumetric clounds (and then removing them because of Oculus rendering issues)
- Adding post processing for auto exposure
- Adding exponential height fog
- Adding areas for foliage
- Creating procedural foliage from Quixel models
- Painting Foliage

A detailed look at the steps and settings I took can be found in [this post](https://voxels.github.io/rebuilding-a-vr-landscape).

Here's a demo of the fully painted background in a capture with hand tracking from an Oculus headset:

{% include youtubePlayer.html id="mSR5ATSRPwg" %}

And here's a sequence rendered to a 4K camera attached to a timeline:

{% include youtubePlayer.html id="PoAqnRDY94U" %}

To render out the sequence to VR, I used the panoramic capture tool to stereo images and took those into Premier Pro to render for Youtube.

At 6K resolution with 16 bit color depth stereo images, my laptop exported about 8 seconds of video frames every 12 hours. For a 90 second video, that's a 4 day render time.  At 4K resolution with 8 bit color depth stereo images, my laptop exported about 2 seconds of video in 7 hours.  I had about 13 seconds overnight, so here is a clip of a 360 video uploaded to youtube:

{% include youtubePlayer.html id="BTBjqj57Iu4" %}

Upgrading to the new Nvidia drivers caused this render process to crash.  I'm going to pick back up after porting this project over to UE5.
