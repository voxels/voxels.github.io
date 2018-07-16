---
layout: post
title:  "Eliminating Collection View Tearing with XCode's Time Profiler Instrument"
date:   2018-07-15 22:30:00 -0000
---

**TL;DR**: *Using the Time Profiler to refactor collection view cell model image fetching and pull Parse's return call off the main thread enables [smooth scrolling]()*.

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

<img src="https://s3.amazonaws.com/com-federalforge-repository/public/resources_portfolio/posts/toyphotogallery_instruments/aws_before_reconverting.png" width="320" alt="AWS Before Reconverting">

The file sizes actually went up by a factor of **5x** after reconverting:

<img src="https://s3.amazonaws.com/com-federalforge-repository/public/resources_portfolio/posts/toyphotogallery_instruments/aws_after_reconverting.png" width="320" alt="AWS After Reconverting">

This should, in theory, slow things down even more because the download sizes are significantly larger, but it's worth it because
my thumbnails looked a lot better:

<img src="https://s3.amazonaws.com/com-federalforge-repository/public/resources_portfolio/posts/toyphotogallery_instruments/high_quality_thumbnails.png" width="320" alt="Optimized Thumbnails">

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

<img src="https://s3.amazonaws.com/com-federalforge-repository/public/resources_portfolio/posts/toyphotogallery_instruments/toyphotogallery_instruments_v1.0.png" width="320" alt="v 1.0 Instruments">

The heaviest stack trace in this run was rendering the JPEG images:
```
  28  1424.0  ToyPhotoGallery (2497) :0
  27  813.0  Main Thread  0x11afef :0
  26 libdyld.dylib 764.0  start
  25 ToyPhotoGallery 764.0  main /Users/voxels/Documents/ToyPhotoGallery/src/ToyPhotoGallery/ToyPhotoGallery/View/Gallery/GalleryCollectionViewImageCell.swift:14
  24 UIKit 763.0  UIApplicationMain
  23 GraphicsServices 759.0  GSEventRunModal
  22 CoreFoundation 759.0  CFRunLoopRunSpecific
  21 CoreFoundation 759.0  __CFRunLoopRun
  20 CoreFoundation 432.0  __CFRunLoopDoObservers
  19 CoreFoundation 432.0  __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__
  18 QuartzCore 427.0  CA::Transaction::observer_callback(__CFRunLoopObserver*, unsigned long, void*)
  17 QuartzCore 427.0  CA::Transaction::commit()
  16 QuartzCore 427.0  CA::Context::commit_transaction(CA::Transaction*)
  15 QuartzCore 381.0  CA::Layer::prepare_commit(CA::Transaction*)
  14 QuartzCore 380.0  CA::Render::prepare_image(CGImage*, CGColorSpace*, unsigned int, double)
  13 QuartzCore 380.0  CA::Render::copy_image(CGImage*, CGColorSpace*, unsigned int, double, double)
  12 QuartzCore 380.0  CA::Render::create_image(CGImage*, CGColorSpace*, unsigned int, double)
  11 QuartzCore 379.0  CA::Render::(anonymous namespace)::create_image_from_image_provider(CGImage*, CGImageProvider*, CGColorSpace*, unsigned int)
  10 ImageIO 374.0  IIOImageProviderInfo::CopyImageBlockSetWithOptions(void*, CGImageProvider*, CGRect, CGSize, __CFDictionary const*)
   9 ImageIO 374.0  IIOImageProviderInfo::copyImageBlockSetWithOptions(CGImageProvider*, CGRect, CGSize, __CFDictionary const*)
   8 ImageIO 374.0  AppleJPEGReadPlugin::CopyImageBlockSetProc(void*, CGImageProvider*, CGRect, CGSize, __CFDictionary const*)
   7 ImageIO 374.0  AppleJPEGReadPlugin::copyImageBlockSet(InfoRec*, CGImageProvider*, CGRect, CGSize, __CFDictionary const*)
```

About 58 percent of the weight was put onto the main thread.  Loading completed around 20 seconds after launch.  The second heaviest weight was located in the ResourceModelController:

```
  34  1424.0  ToyPhotoGallery (2497) :0
  33  813.0  Main Thread  0x11afef :0
  32 libdyld.dylib 764.0  start
  31 ToyPhotoGallery 764.0  main /Users/voxels/Documents/ToyPhotoGallery/src/ToyPhotoGallery/ToyPhotoGallery/View/Gallery/GalleryCollectionViewImageCell.swift:14
  30 UIKit 763.0  UIApplicationMain
  29 GraphicsServices 759.0  GSEventRunModal
  28 CoreFoundation 759.0  CFRunLoopRunSpecific
  27 CoreFoundation 759.0  __CFRunLoopRun
  26 CoreFoundation 110.0  __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__
  25 libdispatch.dylib 110.0  _dispatch_main_queue_callback_4CF$VARIANT$mp
  24 libdispatch.dylib 107.0  _dispatch_client_callout
  23 libdispatch.dylib 106.0  _dispatch_call_block_and_release
  22 ToyPhotoGallery 102.0  _T0Ieg_IeyB_TR /Users/voxels/Documents/ToyPhotoGallery/src/ToyPhotoGallery/ToyPhotoGallery/Model/Utility/SynchronizedArray.swift:0
  21 ToyPhotoGallery 64.0  closure #1 in closure #2 in NetworkSessionInterface.fetchWithAWS(filename:completion:) /Users/voxels/Documents/ToyPhotoGallery/src/ToyPhotoGallery/ToyPhotoGallery/Network/NetworkSessionInterface.swift:104
  20 ToyPhotoGallery 46.0  partial apply for closure #1 in closure #1 in closure #1 in closure #1 in ResourceModelController.build<A>(using:for:with:timeoutDuration:) /Users/voxels/Documents/ToyPhotoGallery/src/ToyPhotoGallery/ToyPhotoGallery/Model/Resource/ResourceModelController.swift:0
  19 ToyPhotoGallery 46.0  closure #1 in closure #1 in closure #1 in closure #1 in ResourceModelController.build<A>(using:for:with:timeoutDuration:) /Users/voxels/Documents/ToyPhotoGallery/src/ToyPhotoGallery/ToyPhotoGallery/Model/Resource/ResourceModelController.swift:0
  18 ToyPhotoGallery 46.0  specialized closure #1 in closure #1 in closure #1 in closure #1 in ResourceModelController.build<A>(using:for:with:timeoutDuration:) /Users/voxels/Documents/ToyPhotoGallery/src/ToyPhotoGallery/ToyPhotoGallery/Model/Resource/ResourceModelController.swift:75
  17 ToyPhotoGallery 46.0  UIImage.__allocating_init(data:) /Users/voxels/Documents/ToyPhotoGallery/src/ToyPhotoGallery/ToyPhotoGallery/Model/Resource/ResourceModelController.swift:0
  16 ToyPhotoGallery 46.0  @nonobjc UIImage.init(data:) /Users/voxels/Documents/ToyPhotoGallery/src/ToyPhotoGallery/ToyPhotoGallery/Model/Resource/ResourceModelController.swift:0
```

About 17 percent of the weight was located in Parse:
16.7% is tied up in Parse
```
123.00 ms   16.7% 0 s    _dispatch_worker_thread3  0x11b020
123.00 ms   16.7% 0 s     _pthread_wqthread
118.00 ms   16.0% 0 s      _dispatch_worker_thread3
118.00 ms   16.0% 0 s       _dispatch_root_queue_drain
115.00 ms   15.6% 0 s        _dispatch_queue_override_invoke$VARIANT$mp
115.00 ms   15.6% 0 s         _dispatch_client_callout
115.00 ms   15.6% 0 s          _dispatch_call_block_and_release
115.00 ms   15.6% 0 s           __55-[BFTask continueWithExecutor:block:cancellationToken:]_block_invoke
113.00 ms   15.3% 0 s            __28-[PFAsyncTaskQueue enqueue:]_block_invoke_2
113.00 ms   15.3% 0 s             -[BFTaskCompletionSource trySetResult:]
113.00 ms   15.3% 0 s              -[BFTask trySetResult:]
113.00 ms   15.3% 0 s               -[BFTask runContinuations]
113.00 ms   15.3% 0 s                -[BFExecutor execute:]
113.00 ms   15.3% 0 s                 __29+[BFExecutor defaultExecutor]_block_invoke_2
113.00 ms   15.3% 0 s                  __55-[BFTask continueWithExecutor:block:cancellationToken:]_block_invoke
113.00 ms   15.3% 0 s                   __62-[BFTask continueWithExecutor:successBlock:cancellationToken:]_block_invoke
113.00 ms   15.3% 0 s                    __56-[PFCurrentInstallationController getCurrentObjectAsync]_block_invoke.40
113.00 ms   15.3% 0 s                     +[PFObject object]
113.00 ms   15.3% 0 s                      +[PFObject subclassingController]
113.00 ms   15.3% 0 s                       -[PFCoreManager objectSubclassingController]
113.00 ms   15.3% 0 s                        _dispatch_queue_barrier_sync_invoke_and_complete
113.00 ms   15.3% 0 s                         _dispatch_client_callout
113.00 ms   15.3% 0 s                          __44-[PFCoreManager objectSubclassingController]_block_invoke
113.00 ms   15.3% 0 s                           -[PFObjectSubclassingController scanForUnregisteredSubclasses:]
```

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
```
  45  861.0  ToyPhotoGallery (2769) :0
  44  361.0  Main Thread  0x14aa88 :0
  43 libdyld.dylib 303.0  start
  42 ToyPhotoGallery 303.0  main /Users/voxels/Documents/ToyPhotoGallery/src/ToyPhotoGallery/ToyPhotoGallery/View/Gallery/GalleryCollectionViewImageCell.swift:14
  41 UIKit 303.0  UIApplicationMain
  40 GraphicsServices 298.0  GSEventRunModal
  39 CoreFoundation 298.0  CFRunLoopRunSpecific
  38 CoreFoundation 298.0  __CFRunLoopRun
  37 CoreFoundation 89.0  __CFRunLoopDoBlocks
  36 CoreFoundation 89.0  __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__
  35 UIKit 89.0  __34-[UIApplication _firstCommitBlock]_block_invoke_2
  34 QuartzCore 87.0  CA::Transaction::commit()
  33 QuartzCore 87.0  CA::Context::commit_transaction(CA::Transaction*)
  32 QuartzCore 80.0  CA::Layer::layout_if_needed(CA::Transaction*)
  31 QuartzCore 80.0  -[CALayer layoutSublayers]
  30 UIKit 79.0  -[UIView(CALayerDelegate) layoutSublayersOfLayer:]
  29 UIKit 77.0  -[UILayoutContainerView layoutSubviews]
  28 UIKit 77.0  -[UINavigationController __viewWillLayoutSubviews]
  27 UIKit 77.0  -[UINavigationController _startDeferredTransitionIfNeeded:]
  26 UIKit 77.0  -[UINavigationController _startTransition:fromViewController:toViewController:]
  25 UIKit 59.0  -[UINavigationController _updateScrollViewFromViewController:toViewController:]
  24 UIKit 59.0  -[UIViewController loadViewIfRequired]
  23 UIKit 58.0  -[UIViewController loadView]
  22 UIKit 58.0  -[UIViewController _loadViewFromNibNamed:bundle:]
  21 UIKit 58.0  -[UINib instantiateWithOwner:options:]
  20 UIKit 55.0  -[UINibDecoder decodeObjectForKey:]
  19 UIKit 55.0  UINibDecoderDecodeObjectForValue
  18 UIKit 55.0  UINibDecoderDecodeObjectForValue
  17 UIKit 55.0  -[UIRuntimeConnection initWithCoder:]
  16 UIKit 55.0  -[UINibDecoder decodeObjectForKey:]
  15 UIKit 55.0  UINibDecoderDecodeObjectForValue
  14 UIKit 41.0  -[UIActivityIndicatorView initWithCoder:]
  13 UIKit 40.0  -[UIActivityIndicatorView _feedTheGear]
  12 UIKit 40.0  +[UIActivityIndicatorView _loadResourcesForStyle:]
  11 UIKit 39.0  _UIImageWithName
  10 UIKit 39.0  _UIImageWithNameAndTraitCollection
   9 UIKit 34.0  -[_UIAssetManager imageNamed:withTrait:]
   8 UIKit 34.0  -[_UIAssetManager imageNamed:scale:gamut:layoutDirection:idiom:userInterfaceStyle:subtype:cachingOptions:sizeClassPair:attachCatalogImage:]
   7 UIKit 34.0  __139-[_UIAssetManager imageNamed:scale:gamut:layoutDirection:idiom:userInterfaceStyle:subtype:cachingOptions:sizeClassPair:attachCatalogImage:]_block_invoke
   6 CoreUI 31.0  -[CUICatalog namedLookupWithName:scaleFactor:deviceIdiom:deviceSubtype:displayGamut:layoutDirection:sizeClassHorizontal:sizeClassVertical:]
   5 CoreUI 31.0  -[CUICatalog _namedLookupWithName:scaleFactor:deviceIdiom:deviceSubtype:displayGamut:layoutDirection:sizeClassHorizontal:sizeClassVertical:]
   4 CoreUI 30.0  -[CUICatalog _resolvedRenditionKeyForName:scaleFactor:deviceIdiom:deviceSubtype:displayGamut:layoutDirection:sizeClassHorizontal:sizeClassVertical:memoryClass:graphicsClass:graphicsFallBackOrder:withBaseKeySelector:]
   3 CoreUI 30.0  -[CUICatalog _resolvedRenditionKeyFromThemeRef:withBaseKey:scaleFactor:deviceIdiom:deviceSubtype:displayGamut:layoutDirection:sizeClassHorizontal:sizeClassVertical:memoryClass:graphicsClass:graphicsFallBackOrder:iconSizeIndex:]
   2 CoreUI 18.0  -[CUIStructuredThemeStore _canGetRenditionWithKey:isFPO:lookForSubstitutions:]
   1 CoreUI 8.0  -[CUICommonAssetStorage renditionInfoForIdentifier:]
   0 libobjc.A.dylib 5.0  objc_msgSend
```

The scrolling still seemed to be pausing, so I did some more digging and found out that even though Parse was being called on a background thread...

<img src="https://s3.amazonaws.com/com-federalforge-repository/public/resources_portfolio/posts/toyphotogallery_instruments/parse_find_sent_on_background_thread.png" width="320" alt="Parse called on a background thread">

...it was in fact returning on the main thread:

<img src="https://s3.amazonaws.com/com-federalforge-repository/public/resources_portfolio/posts/toyphotogallery_instruments/parse_returns_on_main_thread.png" width="320" alt="Parse Returning on the main thread">

This is no good because we had more model work to do on a background thread before trying to finish launch.  
Debugging the DispatchQueues for parse and adding some safety around when to fetch and how to update the collection view
resulted in a much better Time Profiler trace:

<img src="Refactored Time Profiler Trace with No Scrolling" width="320" alt="https://s3.amazonaws.com/com-federalforge-repository/public/resources_portfolio/posts/toyphotogallery_instruments/toyphotogallery_instruments_v1.1_noscroll.png">

In version 1.1, the launch completes in under **8 seconds**.  We only have 40 percent of our work happening on the main thread, and
the heaviest stack trace is now just starting up our error handler.

For good measure, I ran some traces in the [Allocations](https://help.apple.com/instruments/mac/current/#/dev7b8f6eb6), [Leaks](https://help.apple.com/instruments/mac/current/#/dev022f987b), and [Zombies](https://help.apple.com/instruments/mac/current/#/dev612e6956) instruments.  No serious issues popped up after refactoring.

<img src="https://s3.amazonaws.com/com-federalforge-repository/public/resources_portfolio/posts/toyphotogallery_instruments/final_refactor_v1.1_leaksinstrument.png" width="320" alt="Leaks Instrument">

The images below show the CPU and Memory dashboards in Xcode showing how well the app performs under a stress test:

<img src="https://s3.amazonaws.com/com-federalforge-repository/public/resources_portfolio/posts/toyphotogallery_instruments/final_refactor_v1.1_cpudashboard.png" width="320" alt="CPU Dashboard">
<img src="https://s3.amazonaws.com/com-federalforge-repository/public/resources_portfolio/posts/toyphotogallery_instruments/final_refactor_v1.1_memorydashboard.png" width="320" alt="Memory Dashboard">

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
