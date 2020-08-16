# Making a camera app in SwiftUI - Episode 1

<!-- statement of empowerment -->
Read this article to learn how you can start building a camera app in Swift and SwiftUI.

<!-- status update -->
We will be starting by making an empty SwiftUI app and adding a viewfinder.

<!-- tool and environment version -->
I will be using Xcode 12 beta 3 on Big Sur Beta 3

<!-- call to action -->
Start a new project in Xcode, choose iOS app, and configure it like this: 

-  Interface: SwiftUI
-  Lifecycle: SwiftUI App
-  Language: Swift
-  Checkmarks: All disabled

## Info.plist

We need to add a camera usage description to our Info.plist

I find that the easiest way to add this is to:

- Right click on the Info.plist
- Choose `Open As > Source Code`
- Paste the following code as the first item under the topmost `<dict>` node

```xml
<key>NSCameraUsageDescription</key>
<string>$(EXECUTABLE_NAME) uses the camera to take pictures</string>
```

iOS would not have let us do anything with the camera unless we describe what we intend to do with it.

While we have the Info.plist file open, we might make additional changes that are appropriate for a camera app.
Find the following existing nodes and make these changes:

**Disable multiple scenes:**

```xml
<key>UIApplicationSceneManifest</key>
<dict>
	<key>UIApplicationSupportsMultipleScenes</key>
	<false/>
</dict>
```

**Disable Rotation**

```xml
<key>UISupportedInterfaceOrientations</key>
<array>
	<string>UIInterfaceOrientationPortrait</string>
</array>
<key>UISupportedInterfaceOrientations~ipad</key>
<array>
	<string>UIInterfaceOrientationPortrait</string>
</array>
```


## Workspace Setup

Open ContentView.swift

```swift
import SwiftUI

struct ContentView: View {
    var body: some View {
    
        Text("Hello, world!").padding()
    }
}
```

This is currently the one view in our app. This is where we will add our viewfinder.

Since SwiftUI previews will not work we may hide the canvas to get extra space. Hit `CMD + Enter` or go to `Editor > Show Editor Only`. 

## Session

Our first step is to add a capture session.

This is the object the system gives us as a central point to coordinate everything we will be doing with the camera.

```swift
import SwiftUI
import AVKit

struct ContentView: View {
    var session = AVCaptureSession()

    var body: some View {
    
        Text("Hello, world!").padding()
    }
}
```


Now we need a reference to a camera. For the time being we will be working with the front camera.

For the sake of clarity we will not be using the `authorizationStatus` and `requestAccess` API, handling cases where the user has refused access to the camera, or catching errors thrown from the call to `addInput`. However, if you plan to release something to the public, handling all those special cases is important.

```swift
struct ContentView: View {

    var session = AVCaptureSession()

    var body: some View {
        
        // START Setting configuration properties
        session.beginConfiguration()
        
        // Get the capture device
        DEVICE : if let frontCameraDevice = AVCaptureDevice.default(
            .builtInWideAngleCamera,
            for: .video,
            position: .front
        ) {

            // Set the capture device
            do {
                try! session.addInput(AVCaptureDeviceInput(device: frontCameraDevice))
            }
        }

        // END Setting configuration properties
        session.commitConfiguration()

        // Start the AVCapture session
        session.startRunning()
        
        return Text("Hello, world!").padding()
    }
}

```

Here we have gotten a reference to the front camera device, wrapped it in a capture device input object, and added it to the session as input. 

We have made calls to `beforeConfiguration` and `commitConfiguration` on our session to tell AVKit we are making changes to the configuration of the session.

Finally, we have called `startRunning` to get AVKit to start running our session.

## Viewfinder

With the basic session configuration out of the way it is time to start working on our viewfinder.

Since there are no camera components for SwiftUI, we will make use of the framework integration between UIKit and SwiftUI.

First, we need a UIView subclass with a special layer type called `AVCaptureVideoPreviewLayer`.

```swift 

class LegacyViewfinder: UIView
{

    // We need to set a type for our layer
    override class var layerClass: AnyClass {
    
        AVCaptureVideoPreviewLayer.self
    }
}

```

Here we see that we can override the class property `layerClass` on our custom UIView `LegacyViewfinder` class. This way the layer that will be backing instances of `LegacyViewfinder` views will be of type `AVCaptureVideoPreviewLayer`. This is what we need to make a viewfinder.

To be able to show this viewfinder in a SwiftUI app we need a custom SwiftUI view that conforms to `UIViewRepresentable`. This will be a simple and basic Framework Integration. No need for custom Coordinators.

We implement `makeUIView` which will create an instance of our `‌LegacyViewfinder` UIKit view. We will leave `updateUIView` for future episodes.

```swift
struct Viewfinder:UIViewRepresentable {
    func makeUIView(context: Context) -> UIView {
    
        LegacyViewfinder()
    }
    
    func updateUIView(_ uiView: UIView, context: Context) {
        
        // TODO: Stay tuned
    }
    
    typealias UIViewType = UIView
}
```

The next step will be connecting our AVSession to our viewfinder.

First, we need to give our SwiftUI ViewFinder a way to have a  reference to this session. We do this by adding a property to our `Viewfinder`:


```swift
struct Viewfinder:UIViewRepresentable {
        
    var session: AVCaptureSession
    
    func makeUIView(context: Context) -> UIView {
    
        LegacyViewfinder()
    }
    
    func updateUIView(_ uiView: UIView, context: Context) {
        
        // TODO: Handle orientation updates
    }
    
    typealias UIViewType = UIView
}
```

Now we need to connect our session to the `AVCaptureVideoPreviewLayer` layer inside our `LegacyViewfinder` view.


```swift 
struct Viewfinder:UIViewRepresentable {
    
    var session: AVCaptureSession
    
    func makeUIView(context: Context) -> UIView {
    
        let legacyView = LegacyViewfinder()
        PREVIEW : if let previewLayer = legacyView.layer as? AVCaptureVideoPreviewLayer {
            previewLayer.session = session
        }
        return legacyView
    }
    
    func updateUIView(_ uiView: UIView, context: Context) {
        
        // TODO: Stay tuned
    }
    
    typealias UIViewType = UIView
}

```

Finally, we update the body of our `ContentView` to show our `Viewfinder` passing in the reference to the session.

```swift
struct ContentView: View {

    var session = AVCaptureSession()
    
    var body: some View {
        
        // START Setting configuration properties
        session.beginConfiguration()
        
        // Get the capture device
        DEVICE : if let frontCameraDevice = AVCaptureDevice.default(
            .builtInWideAngleCamera,
            for: .video,
            position: .front
        ) {

            // Set the capture device
            do {
                try! session.addInput(AVCaptureDeviceInput(device: frontCameraDevice))
            }
        }

        // END Setting configuration properties
        session.commitConfiguration()

        // Start the AVCapture session
        session.startRunning()
        
        return Viewfinder(session: session)
    }
}

```

Now you can run the app and see yourself.

[Download Project](https://github.com/ianleon/BlogCam/tree/Ep1)

Continue: [Adding filters to a SwiftUI camera app - Episode 2](https://github.com/ianleon/Blog/blob/master/episodes/ep2.md)