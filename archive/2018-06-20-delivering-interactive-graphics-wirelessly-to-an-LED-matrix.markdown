---
layout: post
title:  "Delivering Interactive Graphics Wirelessly to an LED Matrix"
date:   2018-06-01 00:00:05 -0000
---

Driving interactive graphics from the framebuffer of a mobile device to a matrix of LEDs has involved optimizing steps in the pipeline on a case-by-case basis rather than trying to get it all to work at the same time.

<!--break-->

There are a number of alternatives when choosing a set of components to drive an LED matrix.  The controllers can be chosen from a variety of embedded devices, Arduino or otherwise, and the strips, such as WS2812B vs APA104, can be selected to optimize for cost, speed, or color.

In one of my first attempts at building with WS2812B strips, I worked with Kirill Shevyakov, Alex Savich, and Fixx Invictus to construct some columns housing WS2812B strips controlled through a wired connection to a [FadeCandy](https://github.com/scanlime/fadecandy/).

<img src="https://s3.amazonaws.com/com-federalforge-repository/ResonanceMirror/Components/driver/archive/DSC08455.jpg" width="320" alt="Alter">

{% include youtubePlayer.html id="YZmgi2YkHQM" %}

In a follow up project, based on design guidance from Leo Villereal and the other founders of Disorient, we worked with Viktor Getmanchuk and Jason Cipriani to build the Disorient sign for Burningman.

<img src="https://s3.amazonaws.com/com-federalforge-repository/public/artist/matrix_displays/disorient_sign.jpg" width="320" alt="Disorient Sign with Viktor Getmanchuk">

Content for the sign is generated in Processing, and a single Beaglebone Black drives the entire matrix. The Disorient sign has since been adopted by the fine folks at [NYC Resistor](https://www.nycresistor.com).  They provided the really awesome power and communication distribution boards in the initial build.

### Driving LED content from an iOS device

Mobile devices such as the iPhone are fast enough to drive displays like an LED matrix interactively.  One early bottneck to overcome is solved by choosing the right communication protocol and network hardware.  

Color information for a single light is typically four bytes that can be sent to a device communicating via Serial, TCP/IP, UDP or other protocols to a microcontroller driving the LEDs.  Latency can be reduced by working with a fast [router](https://www.ubnt.com/edgemax/edgerouter/).

<img src="https://s3.amazonaws.com/com-federalforge-repository/ResonanceMirror/Components/application/archive/table/DSC08718.JPG" width="320" alt="iPad Driving LED Disk">

Sampling the framebuffer of an iPad rendering content in SceneKit, or any other game engine, allows a user to affect the matrix in real time.  In the image below, an iPad is sending live content to LED rings through a wired connection:

{% include youtubePlayer.html id="riJR49ga8fw" %}

After several attempts of trying out different protocols, I discovered that the MQTT protocol can support delivering brightness values with low latency to a lot of nodes.  In order to test the protocol, I built a coffee table from some white APA104 strips embedded in plexi tubes and diffused with some custom cut plexi panels.  

The content for this table was generated in a SceneKit instance on an iPad, sampled from the framebuffer, sent wirelessly to a router connected through ethernet to a Teensy 3.2 microcontroller.  

In the video below, the iPad's test software drives the diffused LED matrix. 

{% include youtubePlayer.html id="pguY782UVUk" %}

The coffee table was a proof of concept for using an iPad to drive ambient displays.  This project will fold back in to an exploration of [volumetric displays](https://voxels.github.io/iterating-the-design-of-a-volumetric-display) in order to drive the displays wirelessly from roaming depth cameras such as those in the iPhone X.

{% include youtubePlayer.html id="-PAhgJRkSvs" %}