---
layout: post
title:  "Detecting Gestures from the Motion of the Body in Relation to Itself"
date:   2018-06-01 00:00:02 -0000
---

[This page is a Work in Progress...]<!--break-->

Measuring the motion of a skeleton's joints in relation to each other generates signals that can be recognized as gestures.

Depth cameras, such as the [Intel RealSense D415](https://www.intel.com/content/www/us/en/architecture-and-technology/realsense-overview.html), can be configured to generate real-time skeleton data.  

{% include youtubePlayer.html id="fnyEgY6qd5I" %}

The Euler line is...

{% include youtubePlayer.html id="yRd-PS_Xkdc" %}

Tracking the change of the vector of the Euler line in a triangle drawn over three skeleton joints measures the change of position.  

{% include youtubePlayer.html id="U4CIL6vd7dI" %}

Tracking the change of the cross product of pairs created by Euler lines measures the change of direction.  

A [Savistsky-Golay](https://www.mathworks.com/help/signal/ref/sgolayfilt.html) filter can help smooth data captured by the depth camera.

A gesture can be defined as the correlation of change among groups of of vectors.

{% include youtubePlayer.html id="f0ei4WOH0wU" %}