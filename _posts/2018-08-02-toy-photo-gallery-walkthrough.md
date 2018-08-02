---
layout: post
title:  "Toy Photo Gallery Walkthrough"
date:   2018-08-02 00:00:00 -0000
---

{% include base.html %}

The ToyPhotoGallery code base is an example of native iOS development techniques that satisfy common needs among cloud-backed and resource-heavy user experiences.  The application fetches a manifest of resource locations from Parse; retrieves thumbnail and optimized preview image assets from an S3 bucket; presents the thumbnails in a fluidly scrolling, auto-refreshing collection view; and animates into a child view controller designed to scroll across gesture-backed transformations of high-fidelity preview images.

<!--break-->

<center><iframe width="360" height="240" src="https://www.youtube.com/embed/iXQq7ciXVUc" frameborder="0" allowfullscreen></iframe></center> 

<p>

</p>

**Contents**

- <a href="#DesignPhilosophy">Design Philosophy</a>
- <a href="#LaunchingApp">Launching the App</a>
	- <a href="#FetchingManifest">Fetching the Manifest</a>
	- <a href="#RetrievingResources">Retrieving the Resources</a>
- <a href="#ScrollingCollection">Scrolling and Refreshing the Collection View</a>
- <a href="#TransformingGestures">Transforming with Gestures</a>
- <a href="#UnitTesting">Unit Testing</a>

```



```

### <a name="DesignPhilosophy">Design Philosophy</a>

Very few iOS apps operate independently of some cloud-backed service that require asynchronous calls through intermittent cellular connections.  Frequently, at least one of the services delivers resources, such as images, that need to be optimized in a number of ways to reduce the bandwidth that the resource consumes and the time that a customer waits for the asset to arrive.  When a new feature brief asks for a tightly controlled transition animation from a gallery of images to a preview of images, the brief is also implicitly asking for a host of subsystems that can provide the images with minimized demands on the available system resources.  

Some product designers might be satisfied by a quick prototype that solely focuses on animation techniques, perhaps by using locally stored assets and ignoring the refreshing requirements of the gallery collection view, but the results would then only be representative of a misleading fantasy that lack insight into the challenges developers and designers are obligated to face in order to achieve a fluid experience for the end user both before and during the transition. 

>*In order to build the transition between gallery and preview, one must first carefully handle the backing resource store.*

This article walks through the ToyPhotoGallery infrastructure at a high level to document the work required to satisfy the client brief.


The demo was built over a week of intensive development and a subsequent week of light instrumentation and bugfixing.  A more technical analysis of the application's [v1.0 object map](https://voxels.github.io/codesamples_toyphotogallery_diagram) and an article covering [debugging a threading issue](https://voxels.github.io/eliminating-collection-view-tearing-with-xcode-time-profiler-instrument) with instruments is located on [voxels.github.io](https://voxels.github.io).

### <a name="LaunchingApp">Launching the App</a>

Expecting an initial set of photos to be immediately available after a short launch time is a reasonable user expectation.  Achieving the shortest path to a visible screen that has image assets populated puts specific demands on the launch process to be efficient and fail-safe in the event of no connectivity.  

The ToyPhotoGallery uses a **LaunchController** to ensure that the subsystems required by the app are activated and available before moving past the launch screen.  The launch controller provides a time-out mechanism to abort into a number of different destinations that reflect the state of the device and user account if errors are encountered during launch.

Before launch completes, the controller handles the necessary API keys, which are embedded using [obfuscating code](https://github.com/voxels/ToyPhotoGallery#api-key-obfuscation), that begin the following services:
- non-fatal error handling with [Bugsnag](https://www.bugsnag.com)
- remote store control with [Parse](https://parseplatform.org)
- remote bucket handling with [AWS S3](https://www.google.com/aclk?sa=l&ai=DChcSEwiNgLuq_s7cAhVFXw0KHfxTCFkYABAAGgJxYg&sig=AOD64_1l9qMik9VnO0QwCki9Tpto43zy7Q&q=&ved=2ahUKEwjhiLWq_s7cAhWE3FMKHZZqAk8Q0Qx6BAgEEAI&adurl=)
- network session interfacing using [AWSMobileClient](https://docs.aws.amazon.com/aws-mobile/latest/developerguide/getting-started.html) [URLSession](https://developer.apple.com/documentation/foundation/urlsession)

>*The launch controller is backed by a number of protocols that will allow extending launch control into an arbitrary number of required or optional launch services.*

For example, adding an analytics handler such as [mParticle](https://www.mparticle.com) would simply require adding a few keys and an SDK wrapper class.

The controller begins each launch service; determines if it should wait for a response before allowing launch to proceed; waits for asynchronous responses that confirm each subsystem is available; handles errors encountered during launch; and finally signals a successful or failed launch to the AppDelegate.  The delegate asks the launch controller to complete launch by forwarding into the gallery view controller, or a safety-net view controller that would present some user messaging about session failures or reachability problems.

#### <a name="FetchingManifest">Fetching the Manifest</a>

The launch controller is responsible for the first attempt at fetching the manifest of image resources for the user.  Typically there would be an onboarding component for setting up user accounts that direct to the user's record, however that is not handled in this iteration given the scope of the brief.  In this case, the [Parse API fetches](https://github.com/voxels/ToyPhotoGallery#remote-store) a default number of records that must be significantly fewer than the full manifest of all assets.  Fetching a complete manifest at launch is rarely a viable option in the field.

>*ToyPhotoGallery is built using the ModelViewViewModel design pattern.*

A **ResourceModelController** is used to manage building and refreshing the shared model that contains the data needed to locate image resources.  The resource model controller asks a **RemoteStoreController**, an abstraction for wrapping backend store managers such as Parse, to get a count of the total number of records (so that we know when to quit asking for more).  The controller then asks the remote store to go find the first 20 or so image records. Parse should return a list of image data that includes an identifier, thumbnail URL, and URL for an optimized image asset sized that is compressed for presenting on a HiDPI display panel.

##### Error Handling

If the remote store controller encounters an error while attempting to fetch the resource list, the non-fatal **ErrorHandler** will [intercept the problem](https://github.com/voxels/ToyPhotoGallery#non-fatal-error-handling) and forward it to the analytics service.  

>*ToyPhotoGallery uses **throw** throughout the code base in order to gracefully handle failures that could remove the  perception of quality.*

#### <a name="RetrievingResources">Retrieving the Resources</a>

While images could be served from Parse in a different deployment design, using an alternative service such as custom CMS backed by an AWS S3 bucket is a common representation of how many companies will deploy a storage repository.  For this application, we fill in the need by [uploading optimized image assets](https://github.com/voxels/ToyPhotoGallery#upload-image-assets-to-aws) directly to S3.  

##### Optimizing the Images

Before upload, each image needs to be sized and compressed so that they will arrive on device as fast as possible while retaining a justifiable fidelity.  The images are optimized using the Mozilla [JPEG encoder](https://github.com/kornelski/mozjpeg/releases).  A description of the asset generation and uploading process can be found in the project's [README.md](https://github.com/voxels/ToyPhotoGallery/blob/master/README.md) file.

ToyPhotoGallery only uses two versions of an image asset to satisfy the current brief.  

>*The architecture would easily accommodate a request to include full-sized image assets for some new feature by simply adding a column to the manifest, uploading the image assets, and adding a few keys to the fetch requests.*

The resource model controller waits for a response from the remote store controller containing the list of image assets it needs to go fetch.  Once the callback is returned successfully, the model controller then extracts the information from the manifest using a [generic extraction method](https://github.com/voxels/ToyPhotoGallery#non-fatal-error-handling) and returns an existential container for [generic protocol](https://github.com/voxels/ToyPhotoGallery#generic-protocols) that the model controller can use refer to asset locations.  

>*The ToyPhotoGallery uses generic methods to give future developers the opportunity to add different kinds of content to the platform.*

Given a manifest, the resource model controller can ask a **NetworkSessionInterface** to asynchronously, concurrently collect the thumbnail image assets from the S3 bucket.  A network session interface forwards to either the **AWSBucketHandler**, which is always the case in this instance, or to an URLSession dataTask request as a fallback.  

##### Testing with Progressive JPEGs

Progressive JPEGs offer the possibility of loading an image asset through a stream which should present instant results that increase their fidelity as the image resource becomes fully downloaded.  However, AWS S3 does not support downloading image assets gracefully using a URLSession request.  Testing proved that the AWS S3 SDK was the only way to cleanly download assets from the S3 bucket.  In other deployments, Progressing JPEG image containers and [sophisticated URLSession requests](https://gist.github.com/voxels/994a8fb6082d190f0b3a6bd08fbcdf8d) could deliver a more instantaneous user experience.

ToyPhotoGallery employs a time-limited [DispatchGroup](https://github.com/voxels/ToyPhotoGallery#dispatch-queues-and-operation-queues) and concurrent [DispatchQueue](https://github.com/voxels/ToyPhotoGallery#dispatch-queues-and-operation-queues) to download assets as quickly as possible and push through launch.  We want to avoid locking the user into waiting for image assets to completely download at this stage. We do want to make our best attempt at presenting a completed set of image assets above the fold in the gallery view immediately after launch.

After every launch service has notified the launch controller that they are available, and after the manifest has been fetched with as many of the images resources as possible, the resource model controller will [update its delegate](https://github.com/voxels/ToyPhotoGallery#launch-control-with-dispatchgroup), the launch controller, and the launch controller will present a **GalleryViewController** containing a collection view of the thumbnail images.

<img src="http://www.secretatomics.com/resources/toyphotogallery_3.jpeg" width="320" alt="GalleryViewController Screenshot">

### <a name="ScrollingCollection">Scrolling and Refreshing the Collection View</a>

The GalleryViewController requires a number of optimization so that it can present thumbnail image assets while fluidly scrolling across the available data and seamlessly triggering an asynchronous fetch for additional records when the user approaches the end of the list. 

The **GalleryCollectionViewModel** is responsible for checking how deeply scrolling has progressed into the downloaded list.  It handles a *Timer* limited, retry-request to ask its **GalleryCollectionViewModelDelegate** (in this case the resource model controller) to get more available records, if any are left. To fetch the images, it repeats the process during launch for a different page's manifest pointing to locations that are used to fetch image assets and calculate the deltas and sort the view's data source.  The view model calls its delegate, the view, with a list of index paths that have changed so that the collection view can selectively update its layout.

The collection view model is also responsible for calculating the item size for the **GalleryCollectionViewLayout**, a subclass of *UICollectionViewFlowLayout*, that is used for both the vertical, thumbnail-sized scrolling configuration as well as the horizontal, preview-sized scrolling configuration.  

>*The ToyPhotoGallery could be extended to include a custom layout class providing even more control over the transition's animation, however, for the purposes of this brief, the flow layout parent class was used because it was the quickest path to satisfying the requirements.*

#### Updating the Cell Images

{% include youtubePlayer.html id="iXQq7ciXVUc" %}

Each cell contains two image views for the thumbnail and the preview.  The pair of *UIImageView* cross-fades from the thumbnail view in vertically scrolling mode to the preview view in the horizontally scrolling mode. Using two image assets ensures that the large asset will not bog down the gallery view and the small asset will not limit clarity when swiping through individual, full-width, preview images.  

Collection views scroll more quickly when they reuse dequeued cells.  In v1.0 of the ToyPhotoGallery, each cell was responsible for fetching its own image asset, however, the complicated nature of making requests and clearing requests while scrolling through reusable cells prompted a change in v1.1 to make the collection view model responsible for fetching image assets before providing new cells to the collection view.  

A small fade-in animation is applied after a brief delay to give each cell the appearance of arriving in the collection view with a graceful purpose.

```
DispatchQueue.main.asyncAfter(deadline: .now() + 
	DispatchTimeInterval.seconds(Int(appearance.fadeDuration))) {
    	cell.show(imageView:cell.thumbnailImageView, with:appearance)
}
```

<p>
</p>

Achieving a high frame rate that drops in high-fidelity images as quickly as possible is the setup for achieving a seamless transition to a preview window.  Without determining the mechanisms to create this user flow, a deployed solution could become a disappointment for users before they ever reach the new transition.

### <a name="AnimatingPreview">Animating into the Preview</a>

{% include youtubePlayer.html id="xiR5rvbiRDo" %}

A complicated animation process is kicked off when the GalleryViewController receives a signal that the user has tapped on a gallery thumbnail image.  Rather than pushing to a new, separate view controller, a child view controller containing the preview's button controls seamlessly appears, and the gallery collection view is re-purposed to scroll horizontally across the larger images.

Changing the collection view layout class instead of pushing to a new view controller allows us to depend on Apple's support for custom animations changing the appearance of the cells.  In this iteration, we not only change the layout class but also swap out the collection view for a new instance in order to provide independent animations during the swap between collection views.

<img src="http://secretatomics.com/resources/toyphotogallery_1.jpeg" width="320" alt="GalleryViewController Screenshot">

The animation code is presented below:

{% gist e89587ecfa9a4445bb766e03cf27f120 %}

```


```

1. The method instantiates a new collection view and applies the appearance
2. The method calls an AutoLayout cycle so that the collection's internal *UIScrollView* can layout its subviews
3. The method asks the appearing collection view to scroll to the appropriate item
4. Two CABasicAnimations are composed to shift the alpha of each controller through the cross-fade
5. A CATransaction begins with a custom duration and timing function
6. The callback to remove the disappearing collection view becomes the final callback for the transaction
7. The alpha transformations are added to the collection views
8. The bottom view auto-layout constraint for the preview child view controller is adjusted to its expected height
9. The buttons inside the bottom pop out of the drawer given a CGAffineTransform and alpha shift
10. An AutoLayout cycle is run to push through the drawer's translation animation
11. The method checksif we are headed towards a preview window or not:
- If so, the method animates popping in the new heading label and then triggers the close button to appear with a rotation and alpha shift
- If not, the method hides the close button and shifts the heading label to it original state
12. The transaction is completed

```


```

### <a name="TransformingGestures">Transforming with Gestures</a>

The brief contained a secondary requirement to improve animation across the scroll view.  Changing the scale of the cell using a CAAffineTransform is one of the simplest ways to transform cells using the offset data provided by the scroll view during the implicit gesture recognition.  

{% gist ce00e917533c17ae4202cf2da7ecc429 %}

Using a child view controller for the preview configuration would allow us to use an even more advanced scrolling animation if SceneKit was used as the context for the replicating the collection view.  

<img src="http://secretatomics.com/resources/toyphotogallery_2.jpeg" width="320" alt="GalleryViewController Screenshot">

>*In the next iteration, shader transformations like Gaussian blurs, mesh deformations, and subtle material lighting would punch up the design each cell.*

### <a name="UnitTesting">Unit Testing</a>

The complicated model of the photo gallery is a good candidate for [unit testing](https://github.com/voxels/ToyPhotoGallery#unit-testing), in order to ensure that regressions are not encountered from swapping out SDKs or adding to the model's components.  

