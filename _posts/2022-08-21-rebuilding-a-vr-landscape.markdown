---
layout: post
title:  "Rebuilding a VR Landscape"
date:   2022-08-21 00:00:05 -0000
---

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

Then turn off support armv7 and turn on support armv64.  Turn off Open GL and turn on Vulkan.  Then add a package for oculus mobile devices to Oculus Quest 2 and enable remove oculus signature files from distribution APK.

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

### Add landmass custom brush material
### Adding an ocean component
### Adding a sky sphere blueprint
### Adding volumetric clouds
### Adding exponential height fog
### Adding a post processing volume for auto exposure
### Creating a blend material from Quixel textures
### Creating procedural foliage from Quixel models
