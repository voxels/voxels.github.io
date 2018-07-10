## ToyPhotoGallery Object Map

Author: Michael Edgcumbe
Date: 07/10/18

The design goals of the object map include:
- Provide abstract protocols for common services so that libraries can be swapped if needed
- Hide API interfaces behind model controllers
- Provide launch safety for reachability or other issues
- Provide thread safe access to network fetching and local storage
- Provide robust non-fatal error handling for debugging
- Use the Model View View Model design pattern
- Use abstract models within model controllers so that content can be expanded without rearchitecture
- Offer the ability to swap out views as much as possible
- Provide convenience structures for configuring requests and appearances

The diagram below shows the data flow through the major nodes in the object graph:  

![Diagram](https://s3.amazonaws.com/com-federalforge-repository/public/resources/originals/ToyPhotoGallery_ObjectGraphDiagram.png)

The brief for the exercise was to show a transition between the gallery view and the preview,
which was accomplished in the submitted code, however, the focus of the week's efforts to
build the described architecture has been to show how setting up a robust system can 
accommodate future design goals while *also* satisfying the request.  The smooth transition
would be lost work if the system had to be re-architected for a different request.  For that reason
optimization time was spent around tightening up complex model systems rather than trying to 
achieve clean refresh rates or minimize launch time by using local assets


### Launch Control

The first objects created in the app adopt the **LaunchService** protocol.  LaunchService is 
extended by the following types: 

- **ErrorHandlerDelegate** : a protocol for reporting non-fatal errors to services like Bugsnag
- **RemoteStoreController** : a protocol for managing a remote store API, like Parse
- **BucketHandlerDelegate** : a protocol for managing a storage bucket, like S3
- **LogHandlerDelegate** : a protocol for logging messages to the console or to a third party service

Launch services that need to use a secret API key for configuration use the **Obfuscator** class to decode 
a **LaunchContolKey**.  The Obfuscator class accepts a LaunchControlKey, fetches the enum's binary information
and returns a string of the API key.

During *didFinishLaunching:*, the AppDelegate creates instances of **BugsnagInterface**, **ParseInteface**,
and **AWSBucketHander**.  BugsnagInterface inherits from ErrorHandlerDelegate and implements the Bugsnag
SDK for reporting errors.  ParseInterface inherits from RemoteStoreController and implements the Parse
SDK for accesing a remote store.  AWSBucketHandler implements the BucketHandlerDelegate in order to use
the proprietary network manager provided by Amazon.  

The LogHandlerDelegate is currently adopted by **DebugLogHandler**, which prints messages to the console.

Once the service instances have been created during *didFinishLaunching:*, the AppDelegate then creates 
a **NetworkSessionInterface**, a **ResourceModelController**, and a **LaunchController**. 
The LaunchController is resposible for launching the LaunchService objects.  The controller waits 
for didSucceed or didFail notifications from each required LaunchService before notifying the AppDelegate
that the app has launched or failed to launch.  

If launch is successful, the AppDelegate uses the ResourceModelController to build the foundational model's
data source.  

### Model Architecture

The model object in ToyPhotoGallery is represented by an inheritance from the **Repository** generic protocol 
and the **Resource** protocol.  A Repository is a hashmap for Resource objects.  

The ResourceModelController holds a reference to an **ImageRepository**.  An ImageRepository is a hash map for a 
collection of **ImageResource**.  ImageResource is a type of **ImageSchema**, which adopts the Resource protocol.  
Resource guarantees that an object will have an updatedAt date.  ImageSchema adds the properties found in the 
remote store table for an image in addition to updatedAt (i.e. filename, URL, size, etc.).  The imageRepository
reference holds the dictionary properties fetched from the remote store.  Adopting the Repository generic protocol
means that the ResourceModelController can be extended to handle other types of content


### Model Fetching and Extraction

The LaunchController asks the ResourceModelController to build the ImageRepository when the LaunchService objects
have signaled that they have completed configuring themselves.  The ResourceModelController asks its RemoteStoreController
to find a list of object with a default fetch size using a **RemoteStoreTableMap**.  The RemoteStoreTableMap is an emum
with string values that references the schema on the remote store.  It provides a seam by which the remote store can be 
changed without affecting the resource controller.

The RemoteStoreController accepts the query parameters, converts them to the proprietary Parse SDK objects, exectues the query, and returns an array of dictionaries with string keys and common object types that are retrieved from the proprietary PFObject type.  

If the **FeaturePolice** *waitForImageBeforeLaunching* flag is set, the ResourceModelController's **NetworkSessionInterface** will attempt to fetch the thumbnail image data. for the records returned from Parse.  FeaturePolice defines a list of flags that enable experimental features that are not ready for release.

The ResourceModelController accepts the array of raw object dictionaries, checks the type of the Resource, and forwards
extraction to the class of the Resource type, in this case, ImageResource.  ImageResource uses the **Extractor** struct
to to map the array of raw dictionary objects into and array of ImageResource objects

Once the model has been built, the ResourceModelController will call its **ResourceModelControllerDelegate** to indicate
that the model update has succeeded or failed.  During launch, the delegate points to the LaunchController, which 
then signals to the AppDelegate that the app is ready to move forward.  In return, the AppDelegate asks the. LaunchController to push an instance of **GalleryViewController** onto the stack.


### Gallery Collection View, Model, and Layout

The RootViewController is created by the app during launch after the window's UINavigationController 
has pushed through the launch screen. The LaunchController completes a launch by creating a **GalleryViewModel** with its ResourceModelController, and then assigning the model to a GalleryViewController instance created from the Main storyboard.
When the app pushes to the GalleryViewController, launch is complete.

When a GalleryViewModel is assigned to a GalleryViewController, the view controller refreshes layout with a new instance of a **GalleryCollectionViewLayout** configured with a **FlowLayoutConfiguration** that is set up for a vertically scrolling **GalleryCollectionView**.  The GalleryViewController uses its own GalleryViewModel to create a **GalleryCollectionViewModel** that is responsible for providing the data to the collection view.  

The GalleryCollectionViewModel assigns itself as the **GalleryViewModelDelegate**, which provides size information and allows the collection view model to inform the view model that the index paths have been updated in the data source.  The GalleryCollectionViewModel assigns the GalleryViewModel's ResourceModelController as its **GalleryCollectionViewModelDelegate**, so that it can request paged assets from the resource manager directly.  

The GalleryCollectionViewModel uses a **SynchronizedArray** for thread safe access to an array of **GalleryCollectionViewImageCellModel**, the indiviudal data model for each **GalleryCollectionViewImageCell** in the collection view.  The GalleryCollectionViewImageCellModel adheres to the **GalleryCollectionViewCellModel** protocol so that the GalleryCollectionViewModel could be extended with different types of cell content and the GalleryCollectionView can be agnostic to the types of cells it is displaying. 

### Fetching Paged Data

As the user scrolls through the GalleryCollectionView, the GalleryCollectionViewModel checks if new data is required.  If so, the model sends a request to the GalleryCollectionViewModelDelegate, which is adopted by the ResourceModelController.  The model controller handles the fetch through its RemoteStoreController, extracts the ImageResources into its own ImageRepository, and then hands the new records back to teh GalleryCollectionViewModel so that it can insert the items into its own data source of a GalleryCollectionViewCellModel array.  When the update cycle completes, the GalleryCollectionViewModel informs the GalleryCollectionView, acting as the GalleryViewModelDelegate,that it needs to process new IndexPaths and refresh the cells.

### Fetching Cell Image Data

Each GalleryCollectionViewImageCell is responsible for checking the ImageResource model it received from the GalleryCollectionViewModel for an embedded UIImage.  If either images for the ImageResource model's thumbnail or full-size views is nil, the cell is responsible for fetching the image data.  In retrospect, this is not a great design pattern because of the cell's lifespan.  The cell is often not in view when the asynchronous calls to return data received. This functionality should be moved out into the model controller.

### Updating the Gallery View's Scroll Direction

Since the GalleryCollectionViewLayout is the collection view's UICollectionViewFlowLayoutDelegate, the GalleryCollectionView is set to be the **GalleryCollectionViewLayoutDelegate**, which is the channel by which the signal is sent
to show a **PreviewViewController** as a child of the GalleryViewController and switch the scroll direction of the collection view to horizontal.  The GalleryCollectionViewModel delivers sizing information to the layout class acting in the role of a **FlowLayoutConfigurationSizeDelegate**.  

When the signal is sent to change modes in the GalleryCollectionView from gallery to preview or vice-versa, a new collection view is created via the process above and placed into the view hierarchy.  CoreAnimation transactions are used in combination with the UIView convenience methods for animation to swap out the views.















