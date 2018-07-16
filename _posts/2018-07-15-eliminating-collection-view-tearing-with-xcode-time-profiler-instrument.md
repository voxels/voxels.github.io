---
layout: post
title:  "Eliminating Collection View Tearing with XCode's Time Profiler Instrument"
date:   2018-07-15 22:30:00 -0000
---

**TL;DR**: *Using the Time Profiler to refactor collection view cell model image fetching and pull Parse's return call off the main thread enables [smooth scrolling]()*.

<!--break-->

Dealing with complicated code paths involving DispatchQueue can lead to mistakes 
causing the main thread to be blocked when it shouldn't be.

In the video below, the 1.0 version of a toy photo gallery shows that scrolling is not as smooth
as it might be in the gallery view, and we can outpace loading the images in the horizontal preview scroll.

{% include youtubePlayer.html id="SjJ66Wl4tI8" %}

### Resize the Assets Before Doing Anything Else...

I also was unhappy with the compression and resizing of the images in the thumbnails in version 1.0.
Before checking into XCode's [Time Profiler](https://help.apple.com/instruments/mac/current/#/dev44b2b437) instrument, 
I optimized my collection view image cells and the assets stored on S3.  I wanted to make sure that the change in the 
lift would be measured between the unmodified code using the newly sized assets and the final refactoring for v1.1.

I determined the correct thumbnail size using [iosres.com](http://iosres.com). Our largest thumbnails will potentially take up 2/3 of the width of the screen.  The logical width of our largest iPhone screen is 414.  414 * 2 / 3 == 276 points.  
Our images need to be @3x scale to fill each pixel on that width, so 276 * 3 = 828 pixels wide.  

I tried out PNG assets using [sips](http://osxdaily.com/2013/01/11/converting-image-file-formats-with-the-command-line-sips/),
but the resulting image sizes were far too large.
```for i in *.jpeg; do sips -s format png $i --out Converted/$i.png;done```

Then I tried using lossless JPEG compression using some guides from the [Mozilla JPEG Encoder Project](https://github.com/mozilla/mozjpeg)
and [CJPEG Examples](https://calendar.perfplanet.com/2014/mozjpeg-3-0/) and [Compressing JPEG images](https://www.progville.com/frontend/optimizing-jpeg-images-mozjpeg/).

The script below gave me optimised assets:

```
INDEX=0
for FILENAME in *.jpeg; do
  echo $FILENAME
  mozjpegtran -optimise $FILENAME > "$FILENAME.optimized"
done
```

The images looked good, but the file sizes are not that small, so I ended up changing my script to use a compression quality of 85:

**Final JPEG Compression Script**
```
INDEX=0
for FILENAME in *.jpeg; do
  echo $FILENAME
  mozcjpeg -baseline -quant-table 2 -quality 85 -outfile "./converted/$FILENAME" $FILENAME
done
```

The image below shows the thumbnail image sizes before I made them 828 pixels wide and output them with a higher compression quality:

<img src="https://s3.amazonaws.com/com-federalforge-repository/public/resources_portfolio/posts/toyphotogallery_instruments/aws_before_reconverting.png" width="640" alt="AWS Before Reconverting">

The file sizes actually went up by a factor of **5x** after reconverting:

<img src="https://s3.amazonaws.com/com-federalforge-repository/public/resources_portfolio/posts/toyphotogallery_instruments/aws_after_reconverting.png" width="640" alt="AWS After Reconverting">

This should, in theory, slow things down even more because the download sizes are significantly larger, but it's worth it because
my thumbnails looked a lot better:

<img src="https://s3.amazonaws.com/com-federalforge-repository/public/resources_portfolio/posts/toyphotogallery_instruments/high_quality_thumbnails.png" width="640" alt="Optimized Thumbnails">

### Breaking out the Time Profiler

I found a few decent reference articles about the Time Profiler:
- [iOS Performance Tutorial from OKCupid](https://tech.okcupid.com/ios-performance-tutorial-from-okcupid/)
- [Using Time Profiler in Instruments](https://developer.apple.com/videos/play/wwdc2016/418/)
- [Time Profile Documentation](https://help.apple.com/instruments/mac/current/#/dev44b2b437)

I used the following procedure to standardize my tests in the Time Profiler:

**Time Profiling Test**
0) Delete the app on the test device
1) Run the app with 'Profile' in Xcode
2) Choose the Time Profiler in Instruments
3) Run the app, without touches, for 40 seconds
4) Stop and save

For comparison, I did not start testing with any scrolling.  I just wanted to see what fell out of the data
with no interaction at all during launch.  So, with the unmodified code in version 1.0 using the new asset sizes, 
I got the following results in the Time Profiler:

<img src="https://s3.amazonaws.com/com-federalforge-repository/public/resources_portfolio/posts/toyphotogallery_instruments/toyphotogallery_instruments_v1.0.png" width="640" alt="v 1.0 Instruments">

The heaviest stack trace in this run was rendering the JPEG images:

{% gist a7382d1146de6694a7b060abc788a1bb %}

About 58 percent of the weight was put onto the main thread.  Loading completed around 20 seconds after launch.  The second heaviest weight was located in the ResourceModelController:

{% gist 7931263db0250766e47b4f641fac3a74 %}

About 17 percent of the weight was located in Parse:
16.7% is tied up in Parse

{% gist e96ee476379680a445cb22192fcc0c2a %}

Given the results above, it seemed like I needed to do some refactoring of the image cell model, the image cell view, and check into Parse.

### Refactoring for Better Performance

I found some tutorials online that helped me understand some [gotchas](https://medium.com/capital-one-developers/smooth-scrolling-in-uitableview-and-uicollectionview-a012045d77f) about collection views. 

In order to improve the collection view scrolling, I did some work on the following:
1) Move fetching and configuration out of the cell
2) Avoid UIColor clear and shadows.
3) Simplify the gallery collection view controller model to use ImageResources directly instead of a wrapper 
for a generic cell model.
4) Audit the dispatchqueues

Emptying the images out of the cell's views changed the heaviest stack trace:

{% gist 44a81bcff2d684a7bcd400751cb1cb37 %}

The scrolling still seemed to be pausing, so I did some more digging and found out that even though Parse was being called on a background thread...

<img src="https://s3.amazonaws.com/com-federalforge-repository/public/resources_portfolio/posts/toyphotogallery_instruments/parse_find_sent_on_background_thread.png" width="640" alt="Parse called on a background thread">

...it was in fact returning on the main thread:

<img src="https://s3.amazonaws.com/com-federalforge-repository/public/resources_portfolio/posts/toyphotogallery_instruments/parse_returns_on_main_thread.png" width="640" alt="Parse Returning on the main thread">

This is no good because we had more model work to do on a background thread before trying to finish launch.  
Debugging the DispatchQueues for parse and adding some safety around when to fetch and how to update the collection view
resulted in a much better Time Profiler trace:

<img src="Refactored Time Profiler Trace with No Scrolling" width="640" alt="https://s3.amazonaws.com/com-federalforge-repository/public/resources_portfolio/posts/toyphotogallery_instruments/toyphotogallery_instruments_v1.1_noscroll.png">

In version 1.1, the launch completes in under **8 seconds**.  We only have 37 percent of our work happening on the main thread.

For good measure, I ran some traces in the [Allocations](https://help.apple.com/instruments/mac/current/#/dev7b8f6eb6), [Leaks](https://help.apple.com/instruments/mac/current/#/dev022f987b), and [Zombies](https://help.apple.com/instruments/mac/current/#/dev612e6956) instruments.  No serious issues popped up after refactoring.

<img src="https://s3.amazonaws.com/com-federalforge-repository/public/resources_portfolio/posts/toyphotogallery_instruments/final_refactor_v1.1_leaksinstrument.png" width="640" alt="Leaks Instrument">

The images below show the CPU and Memory dashboards in Xcode showing how well the app performs under a stress test:

<img src="https://s3.amazonaws.com/com-federalforge-repository/public/resources_portfolio/posts/toyphotogallery_instruments/final_refactor_v1.1_cpudashboard.png" width="640" alt="CPU Dashboard">
<img src="https://s3.amazonaws.com/com-federalforge-repository/public/resources_portfolio/posts/toyphotogallery_instruments/final_refactor_v1.1_memorydashboard.png" width="640" alt="Memory Dashboard">

CPU time spikes at launch and then remains pretty modest.  Memory usage never climbs above 35 Mb even after all the images are loaded
and lots of high intensity scrolling is thrown at the gallery and preview collection views.

### Comparing v1.0 and v1.1

In version 1.0, the toy photo gallery demonstrated pauses during scrolling, unoptimized thumbnails, and unrefined transition animations:

{% include youtubePlayer.html id="SjJ66Wl4tI8" %}

{% include youtubePlayer.html id="AzOr3a0rP64" %}

In version 1.1, the photo gallery has smooth scrolling, optimized thumbnails, and refined transition animations:

{% include youtubePlayer.html id="-qtwFixZq9I" %}

{% include youtubePlayer.html id="xiR5rvbiRDo" %}

Overall the scrolling performance is much better after using the Time Profiler:

{% include youtubePlayer.html id="iXQq7ciXVUc" %}
