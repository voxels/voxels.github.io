---
layout: post
title:  "Generating Interactive Particle Systems in OpenCV and SpriteKit"
date:   2018-06-01 00:00:03 -0000
---

This project is a proof of concept for capturing infrared video from a homebrew camera, sending the frames through OpenCV to generate keypoints and contours, and then forwarding that data into SpriteKit as the source vector for particle systems and other visual highlights.  The video analysis and particles are processed in real-time on device.

<!--break-->

The links below show the components used for a homebrew IR camera.  In order to make the camera, one just cracks open the webcam and slips in a slice of the IR filter.  The light helps create contrast that becomes helpful for analysis in OpenCV.

- [Webcam (Amazon)](https://www.amazon.com/gp/product/B0080CE5M4/ref=oh_aui_search_detailpage?ie=UTF8&psc=1)
- [Lighting (Amazon)](https://www.amazon.com/gp/product/B00NFNJ7FS/ref=oh_aui_detailpage_o00_s00?ie=UTF8&psc=1)
- [Filter (B&H Photo)](https://www.bhphotovideo.com/c/product/292664-REG/LEE_Filters_87CP3_3_x_3_Infrared.html)

The video below shows an OpenCV preview of the homebrew camera generating keypoints and contours on OpenCV.

{% include youtubePlayer.html id="4q2MzWvaNBI" %}

Extracting a single channel of color information as grayscale data can help speed up the video analysis.  The code below shows how some low level APIs send the data into OpenCV and then on to an operation queue used by another component in the application.

{% gist 96efadcdf4e5b178a3bf20cc278dbb80 %}

The video below shows the extracted keypoints and contours generated in SpriteKit:

{% include youtubePlayer.html id="6stWyq5aPUg" %}

Once the data is available, SpriteKit effects, such as particle systems, create nice effects.

{% include youtubePlayer.html id="3VCu45F0iUU" %}

This rendering was projected back into the space being captured at an art event on Chelsea Piers.