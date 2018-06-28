---
layout: post
title:  "Generating Interactive Particle Systems in OpenCV and SpriteKit"
date:   2018-06-01 00:00:03 -0000
---

[This page is a Work in Progress...]<!--break-->

Infrared cameras produce color information that is consumed by OpenCV.

- [Webcam (Amazon)](https://www.amazon.com/gp/product/B0080CE5M4/ref=oh_aui_search_detailpage?ie=UTF8&psc=1)
- [Lighting (Amazon)](https://www.amazon.com/gp/product/B00NFNJ7FS/ref=oh_aui_detailpage_o00_s00?ie=UTF8&psc=1)
- [Filter (B&H Photo)](https://www.bhphotovideo.com/c/product/292664-REG/LEE_Filters_87CP3_3_x_3_Infrared.html)

{% include youtubePlayer.html id="4q2MzWvaNBI" %}

Color channels can be quickly reduced to grayscale data.

{% gist 96efadcdf4e5b178a3bf20cc278dbb80 %}

OpenCV generates keypoints and contours.

{% include youtubePlayer.html id="6stWyq5aPUg" %}

SpriteKit can deliver real-time particle systems using the OpenCV data.

{% include youtubePlayer.html id="3VCu45F0iUU" %}