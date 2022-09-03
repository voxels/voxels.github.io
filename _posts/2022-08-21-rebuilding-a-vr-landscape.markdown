---
layout: post
title:  "Rebuilding a VR Landscape"
date:   2022-08-21 00:00:05 -0000
---

I took the steps below to set up a landscape environment for VR:

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

<!--break-->


### Looking at the landscape on Oculus Quest 2

I've been working on transferring the VR pawn from the VRTemplate sample project into my own projects.  I'm starting off by rendering the basic third person shooter sample project in VR to find out what is contained in the template's blueprints, and then I want to drop the VR pawn into my virtual landscape to see if the Oculus can render the Quixel assets (in the sample projects it seems like they maybe can't.)

I learned along the way that the docs for Oculus are wrong..  you need to use VS 2019 instead of 2017, and I also learned that you can't run the Oculus repo from an external USB drive or copy it from one place to another.  The only way it builds and runs is to clone the repo to the internal drive and build with VS 2019.  

Because of a problem with Microsoft OneDrive, I lost the edits I made to my original landscape for a VR demo.  I figure it's OK because I didn't document the steps I took to get something I liked and doing the process over again would help me remember the things I learned.

Below are some screenshots and recordings of each step I took.

### Importing the heightmap for the landscape

First, download the heightmap grayscale file from the US government.  USGS National Map Downloader (https://apps.nationalmap.gov/downloader/) is a good resource. I select all the elevation data sets, put an extent on the area I'm trying to download, and select the data sets with the highest resolution.

[USGS Data Source Screenshot]

Then I take the files into an application which will open GeoTiff and export PNG.  The application I use is free and called [QGIS](https://www.qgis.org/en/site/index.html).  It's pretty simple.  You add your tif as a data source from Layer > Data Source Manager and then export to png from File > Import/Export.

From there I take my pngs into Photoshop and join together the landscapes that are next to each other.  There is an issue where the greyscale doesn't match from one side to the other, but rather than blend it out in photoshop, I choose to do that in Unreal.

[Photoshop Elevation Map]

### Creating a new Unreal project

Since I want to work with this project in VR, I'm using the Oculus fork of the Unreal Engine.  Instructions for setting up the engine are on the [Oculus github](https://github.com/Oculus-VR/UnrealEngine). (Requires an epic account to see the repo).

I learned through errors that the Oculus repo will not compile from an external USB drive.  It's also not possible to cleanly copy the repo from the drive to the internal drive.  It only worked for me by compiling in VS 2019 from my internal drive.  It adds several hundred Gb to disk space when it's finished.

After opening the Oculus version of the Unreal Engine, I create a new Games type project, Blank Project Template, No Starter Content and save it as the project name.

### Adding a VR Hand Tracking Pawn

The blank project has some things in it that we don't want.  I delete the atmospheric fog, the floor, and the sphere reflection capture.

I add a landscape using the png I have from above as the heightmap by going to the landscape tool in the modes panel, choosing import from file, enabling edit layers, and adding the heightmap file as the 16-bit greyscale PNG I exported from photoshop of the combined maps.  I increased the number of components to give me an underwater layer around the island.

From there I put the player start camera on top of the highest elevation

[Landscape Player Start Location]

From here I save and close the project.  The pawn I want to use comes out of the VR Handtracking Sample in Oculus Samples.  I closed my project temporarily and opened the handtracking sample from Oculus.  From here I migrate the VRCharacter pawn and then go to the input project settings and export those to a file.  Then I go back to my Unreal project and open the level with my landscape.  I create a new game mode blueprint and set the default pawn class to VRCharacter.  

[Game Mode - VR Character]

Compile and save the blueprint, then open the project settings maps and modes.  Then change the defaut game mode to VRGameMode and the startup map and default map to VRLandscape.

[Project Settings Maps and Modes]

After building the lighting, I hit the play in editor button to see if everything is opening the way I want.  


### Setting up to compile for Oculus

In project settings, there are a number of things to change in order to build for the Oculus.

First, open the plugins and enable Oculus VR plugin if it isn't already.  Then, open the Android Project settings and click the configure button, then change minimum SDK to 29, package game data inside sdk enabled, allow large OBB files enabled. 

[Project Settings - Android SDK]

Then turn off support armv7 and turn on support armv64.  Turn off Open GL and turn on Vulkan.  Then add a package for oculus mobile devices to Oculus Quest 2 and enable remove oculus signature files from distribution APK. Also turn off Mobile HDR and enable forward shading and set the MSAA to 4.

[Project Settings - Build and Packaging]

For a full list of settings to optimize for the Quest, watch this video.


{% include youtubePlayer.html id="y3xFZF9Nyt4" %}

Save the map and settings and then launch to a Quest connected by a USB cable.  Launching the project takes about 10 - 20 minutes at this point.

In order to VR Preview, you have to have the SteamVR plugin open, and the Oculus headset open the Rift app while the headset is connected over USB.

Right away the camera is not tracking the floor, and it's because the level blueprint is not set up. I couldn't find a way to copy from one level blueprint to another, so the only way to fix is it to rebuild the blueprint.  Basically, you set the tracking origin to the floor on begin play, get the player pawn as your VR Character, and set the visibility for the hand, controller, and tracking.  I use the custom tracking hands because they have a mesh.

[Level blueprint]

While testing this, I learned that if you want your hands in front of you, then your Player Start component needs to be oriented at 0,0,0.

Here's a video of the correct hand models and orientation using a streaming video of the Oculus POV:

{% include youtubePlayer.html id="lhIQXFC29EA" %}

Here's a little more of the surrounding landscape after I fixed the terrain layer and put in a floor so I wasn't sliding off the hill:

{% include youtubePlayer.html id="T3f2OLYJNQU" %}

The next step is to smooth out the landscape with the erosion, flatten, noise, and hydro sculpting tools and add a few platforms for VR navigation.

### Adding hand tools to the pawn

I want to use hand controllers with teleportation, and I want to target the platforms for specific viewpoints.  There is an example of teleportation in the Unreal VRTemplate and there is a targeting example in the Oculus HandTrackingTrain sample.

Starting with the targeting, I migrated the windmill, tools manager, and pawn from the Oculus sample to my landscape.

The hand tracking pawn depends on blueprint which has a C++ parent class

First I copied over the C++ source files.

Then I added the following to the uproject file:
	"Plugins": [
		{
			"Name": "OnlineSubsystemOculus",
			"Enabled": true
		}
	]

Then I added the following to the project's Build.cs file
    PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "InputCore",
		"OculusInput"});

	PrivateDependencyModuleNames.AddRange(new string[] { 
			"HeadMountedDisplay", "OculusHMD" });

Then I changed all the module names from HANDSTRAINSAMPLE_API to my new project module name VRLANDSCAPEDEMO_API

I removed HandsTrainSample.h and .cpp to fix a build error about duplicate symbols.  Also the HandsTrainSample.Build.cs.

I changed my game mode to use the HandsTrainingSamplePawn, and I rebuilt the BP_HandsVisualizationSwitcher to a new class with Hands Visualization Switcher parent actor class.

Before reparenting I tried these steps:
- copy the c++ files into the new project
- change the source code accordindly
- Delete: .vs, binaries, build, intermediate, saved, DerivedDataCache, .sln file, .vc.db file
- generate VS project files
- open VS project and rebuild
- if no errors, open UE4 project

I also had to rebuild the HandRaytool, the FingerTipPokeToolIndex, the InteractableToolsManager, HandsVisualizationSwitcher, Windmill, and WindmillCollidableInt.

I exported the inputs and collision channels to files and imported them into my project.

After I rebuilt the classes with the correct parent classes, I added a BP_InteractableToolsManager and a windmill to my scene.  The hands don't usually get detected right away, and the tool doesn't seem to interact with the windmill, so I need to look more into that code.

### Adding teleportation to the motion controllers

Since that part didn't work right away, I switched back to my VRCharacter pawn and migrated the VRPawn from the Unreal VRTemplate project into my project to grab the teleport code.

To test the code, I change my game mode to use the VRPawn character, added a nav mesh bounds volume to the platform my player starts on, and imported the inputs and collision channels to my project.


{% include youtubePlayer.html id="NFnLaZ-ArPA" %}

After testing the VRPawn, I copied all of its functions, variables, and local variables into my VRCharacter class.  That added the teleporting to my hand tracking pawn.

Then I tried adding in the hand tracking events I want to use to effect my teleport.  I followed along with this video to attach the gestures they suggest to the teleport function.

{% include youtubePlayer.html id="fsxH-z8qx58" %}

After tweaking some interactions between the motion controllers and the hand events, I ended up with this:

{% include youtubePlayer.html id="nLHUL4MSlGc" %}

The right hand axes were inverted, so I had to multiply by negative numbers to get the rays to face correctly.

### Adding a vr movement sequence

I wanted to move my vr pawn from one place to another across a smooth spline if it collides with a particular sphere.  While I was trying this out, I learned that adding a second nav mesh to my scene caused the teleport function to often return false, so I changed my nav mesh size for a single mesh to cover the bounds of the whole landscape.  To speed up the navigation generation, I used a [navigation invoker](https://docs.unrealengine.com/4.27/en-US/InteractiveExperiences/ArtificialIntelligence/NavigationSystem/UsingNavigationInvokers/) on my pawn.  Neither of my files wanted to save after I made these changes the first time.  

I also ran into this [bug](https://issues.unrealengine.com/issue/UE-11447) in Unreal.  My pawn was falling through my landscape at the seams of components, so I resamples the components around my player start and rebuilt my project.   That didn't work so I turned off gravity on my movement controller and set the steepest angle to 90 degrees.  Then I stopped falling through the floor.

First I got a collision to move the blueprint sphere itself:

{% include youtubePlayer.html id="KnE-f9WEBAw" %}

And once that was done, I moved the VRCharacter (and preserved it's rotation) along the spline associated with the blueprint instead:

{% include youtubePlayer.html id="3bwVYohLnH8" %}

So from here I have a way to move around my environment and create paths that fly me from one place to the other across the landscape.

### Setting up the landscape with Quixel materials

In order to paint my landscape, I created a material that blends together other materials that I downloaded from Quixel.

This video shows how to create the blend material:

{% include youtubePlayer.html id="X0MN9xWr-iI" %}

Since Quixel materials come with a displacement map, I followed along with this video to add a displacement to my landscape material:

{% include youtubePlayer.html id="6IsNLX321YI" %}

I created my own blend material with 8 different surfaces by connecting up albedo, normal, displacement, and roughness to a layer blend.  Then I went to the landscape paint mode and created my layer info objects.  I got a warning about my texture streaming pool size, so I changed all my texture asset max texture sizes as described [here](https://stefanperales.com/blog/ue5-quick-tip-texture-streaming-pool-over-budget/).  I set all my tiling for each texture and set the displacement value I wanted for each.

Here's an example of the world with a single landscape material:

{% include youtubePlayer.html id="S5dUobaThko" %}

I found a problem painting landscape materials with the Oculus engine that didn't exist in the default 4.27 engine.  My landscape material disappeared when I used the Oculus engine, so I copied my project over to the default engine to keep working.

I also found a problem in the regular engine using more than 8 different kinds of textures in my landscape material.  When I tried to add 10, my landscape stopped rendering my fill layer.  When I took it back to 8, the layer rendered again (even with texture compression).  So rather than fight with it, I chose to limit what I could paint to the landscape to 8.

I chose gravel as my base layer, since it has the least about of patterning at a high level.

### Adding blueprint brushes to the landscape

In order to add some landmass displacement, I enabled the Landmass plugin, went into landscape mode, and added a blueprint brush of type CustomBrush_MaterialOnly.  

In 4.27, the material only brush was giving me an empty blueprint with no components.  At this point, I uninstalled 4.27 and installed UE5 just to confirm it wasn't the engine.  I also wanted to try out my project in UE5 to see if it would convert.  I also converted my project with the fixed material back to the Oculus version of 4.27, to see if the Oculus engine still had a problem.  The landscape rendered correctly, but the landmass blueprint brushes had no components below the root component.  When I enabled the landmass plugin with the Oculus engine, and rebuilt UE4 and the project, they both built successfully, but the project complained that it could not find files in the content folder, so it would not open.  (I made the mistake of trying to convert in place.)

In order to get UE5 to convert the project, I had to install DotNET 3.1.  I also had to add a line to my HandsVisualizationSwitcher.cpp:  	

#include "Components/InstancedStaticMeshComponent.h"

I copied over my content and source folders and reset all the project settings from before in a new UE5 project.  That seemed to make everything work again.

So, now I have my project running in UE5, but I'm still not seeing any components in the blueprint brushes when I drag them out.  To be sure, I tried this in another C++ empty template project.  Same issue there.  For some reason blueprint brushes are broken in all versions right now, or I've missed something by looking at the docs.

When I tried to run the project as a VRPreview in UE5, the editor crashed with a exception related to hand tracking.  So, I was forced to downgrade again, back to the Oculus 4.27 engine, by copying the content and source folders to a new project and resetting all my project settings (inputs, maps and modes, android, oculus VR) and plugins.

I tried the blueprint brushes one more time, carefully stepping through the actions in this [video](https://www.youtube.com/watch?v=GK3KAevUy8E), but when I clicked on the landscape with a landmass blueprint brush, my editor crashed twice with this [error](https://forums.unrealengine.com/t/crash-when-using-tiled-landscape-in-map-h/155300).

Since I couldn't get the blueprint brushes to work, I checked for the crash in VRPreview again in 4.27.  After reloading the engine from scratch and opening my project in 4.27 (not Oculus), the landscape painting and blueprint brushes started to work again.  I finally added the material only brush to add some dimension to the landscape.

I also added in some macro variation textures into the landscape materials to break up some of the flatness.

### Painting the landscape

After adding a a few materials, I tried out the navigation.  It works better on flat surfaces, so I decided to add in a few floors, and move the endpoint of my movement spline to a different vantage point.  Here's a video of all the rock and mud materials, no vegetation, with a view that takes you from the top to the midpoint.

{% include youtubePlayer.html id="3HumfKd3Aa0" %}

### Adding an ocean component

To add water, I enabled the Water plugin and places a Water Body Ocean actor on my scene.  then set the scale of the ocean actor, then the tile extent of the ocean mesh actor, then changed the boundaries of the water to fit around my landscape.  I then had to realign my landscape and other dependent actors in the z axis because my water wasn't showing up below a certain point.

{% include youtubePlayer.html id="r57jR4HAxc4" %}

At this point, I added another two vantage points, so you could see the water up close.

{% include youtubePlayer.html id="3YBNZ7djnrY" %}

### Adding a sky sphere blueprint

The base level came with a sky sphere blueprint, but the sky is missing some dimension.  

I set my sky sphere bp scale to 10 on every axis, and set the horizon and zenith color to black.  I turned off clouds determined by sun position.

I added a sky atmosphere.  I added a sky light and disabled the lower hemisphere is solid color.  I enabled cloud ambient occlusion and changed the ambient occlusion extent.

I set my directional light to moveable.  Checked that atmosphere sun light was enabled.  Enabled cast cloud shadows.  Turned on light shaft occlusion and light shaft bloom.

Then I spent a few minutes placing my sun in the atmosphere.  I wanted to create some contrast between day and night, and specifically create some darkness around the spots I've landed my camera in.


{% include youtubePlayer.html id="CDwHsY9oD7Y" %}

### Adding volumetric clouds

The next thing I added was volumetric clouds.  I tweaked some of the settings to make them more visible.  I noticed that they are moving in a different direction than the background clouds, but I didn't see a setting to change the wind direction, so I left it alone.  I  found that the left and right eye were not rendering the same distance for the clouds.  The left was much shorter.  I set the tracing start max distance to 140 to bring the rendering back to a level where the eyes were cutting the rendering of the clouds around the same point.  At low elevations, one eye was rendering the clouds through the landscape.  I tried to fix this by raising the lower level of the clouds.  But ultimately I could not get the volumetric cloud to render behind the landscape in the right eye, so i removed it from the build.


{% include youtubePlayer.html id="WP69R4nAzZ0" %}


### Adding a post processing volume for auto exposure

I think that the Quest doesn't have post processing based on what I remember from a video, but I'm not certain.  For the preview, I added a post processing volume and set the min and max brightness to 1.0.  I also set the scale and made it an infinite extent.  This shows the model with the post process volume and no volumetric clouds.  

{% include youtubePlayer.html id="aKHbfYgse_M" %}

### Adding exponential height fog

The last bit of atmosphere I added is the exponential height fog.  It blends out at the horizon and adds some depth to your your lower terrain.  I set the fog density and changed the in scattering color to black.  I also turned on my project setting for support sky atmosphere affecting height fog and restarted my project.  When I did this, what seems like all of my shaders recompiled.

I noticed at the end of this video that again one eye was producing a shadow that the other one wasn't.  I chose not to delete the offending shadow because you only see it at the shore line.

{% include youtubePlayer.html id="qNA_xWTV0Zo" %}


### Adding areas for foliage

I painted grassy soil, uncut grass, and ferns across the lower parts of my landscape.  Then I outlined the area in beach pebbles along the shoreline.  I had a problem with some of my components rendering until I checked my material and found some texture samples that were not set to Shared:Wrap.  Fixing that blended all of my landscape materials correctly after my shaders compiled again.


{% include youtubePlayer.html id="oq_9Wp_jALc" %}


### Creating procedural foliage from Quixel models

From here, I want to add in procedural meshes on top of my landscape materials to give the landscape some textures.  I opened my material and added and landscape grass output, with a grass type for each layer in my landscape material.  Then I created landscape layer samples for each layer and connected it to the grass output.  Then I downloaded the material assets I wanted in my foliage from Quixel.

First I set up my GrassySoil landscape grass type with a type for each of the grass meshes plus one taro and one palm.  I set the density at 10 to start with for all, and for all I turned off casts dynamic shadow.

Here's a test of grass types for ferns, wild grass, and taro:

{% include youtubePlayer.html id="t4Gi06U0a24" %}

Tweaking these paramaters created a lot of waiting for shaders to compile.


I found that my boulders were not showing a material, so I deleted the megascans plugin and reinstalled it and deleted the Saved and Intermediates folder and reopened the project.  That didn't solve the problem.  My default master material had none of its nodes hooked up.  So I deleted the plugin again and deleted the plugin source install folder and tried again.  Then I opened a new project and saw I had the same problem.  After following the instructions [here](https://help.quixel.com/hc/en-us/community/posts/360014460617-Exporting-Surfaces-to-Unreal-Engine-Showing-Black-texture-instead-of-actual-texture-), I found that I still had the same problem in a new project.  So, on a whim I tried copying the whole MS_Presets folder from Quixel Megascans plugin folder to the Content folder of my new project.  That fixed the material problem I had.  I migrated the MSPreset folder back into my landscape project.  I only had to fix one of my plant assets to ge tmy foliage to look the way it was when I noticed the problem.  I re-exported the rock megascans assets to my project.  This time the material instances were present and the assets had the right textures on them.

I painted everything but the sand.  I had some nice coral for underwater but my textures were already pushing the limit and I still need to add trees.

{% include youtubePlayer.html id="SOi_58z6bH0" %}

At the end you can see some of my rocks floating in the air.  I unchecked align to surface on that asset and it seemed to put them back on the ground.

### Painting foliage

Quixel does have some good tree assets, if you're doing a european environment, but mine is not.

{% include youtubePlayer.html id="QK5gaeEAig" %}

I went to the marketplace and downloaded some Quixel and paid-for assets for my scene.

I made foliage types for each of my static meshes by dragging the static mesh into the foliage paint tool and painted about about 3-10 meshes per biome in my landscape.

After painting two of the four landscapes, I tested the meshes in VR.  I should have done this earlier.  The frame rate was very low.  I went back through and deleted half my meshes.  I tried it again, and found that hand tracking was not stable even with half the meshes, so I went back in and deleted the meshes in places you could not see from the destinations.  I tried that and my editor crashed.  So, I tried reapplying the meshes with a cull distance that was not 0, and that returned my presentation back to a playable framerate.  I also noticed that the hand tracking lost my hands when they were in direct sunlight, so I closed my curtains and the hand tracking got better.

I decided to opt out of the floor since it was competing with the terrain.  I had to move my paths to new locations to get them closer together so you don't have to go as far.  I tried adding collision to my procedural materials but it didn't work, so rather than looking into it I just moved on.  I had to get used to the editor crashing and recompiling shaders when I moved something, so I tried to remember to save before attempting to play the level.

Here's a demo of the fully painted background with all the paths adjusted:

{% include youtubePlayer.html id="mSR5ATSRPwg" %}


