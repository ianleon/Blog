# Adding filters to a SwiftUI camera app - Episode 2

We will be continuing our SwiftUI camera app by implementing filters on the viewfinder.

I will be using Xcode 12 beta 4 on Big Sur Beta 4.

If you want to code-along you can use the project from the last episode as a starting point.

[EP 1 Project](https://github.com/ianleon/BlogCam/tree/Ep1)

## Get Frames

The first thing we need to do is capture the frames we need to filter. We need to create a frame pipeline to enable this.

First we need to add a special output called `AVCaptureVideoDataOutput` to our session. Then we need a dedicated `DispatchQueue`, and a delegate object that can receive frames. For the time being we will be adding these to our `ContentView`.

```swift
struct ContentView: View { 
    // ...
    
    var framesOut = AVCaptureVideoDataOutput()
    let framesQueue = DispatchQueue(
        label: "com.ianleon.blogcam",
        qos: .userInitiated,
        attributes: [],
        autoreleaseFrequency: .workItem
    )
    let framesDelegate = FramesDelegate()
    
    // ...
}
```


This is what the `FramesDelegate` looks like right now. Notice the `AVCaptureVideoDataOutputSampleBufferDelegate` protocol being used. The prints will confirm that the frames are being sampled.

```swift
class FramesDelegate:NSObject,AVCaptureVideoDataOutputSampleBufferDelegate {
    func captureOutput(
        _ output: AVCaptureOutput,
        didOutput sampleBuffer: CMSampleBuffer,
        from connection: AVCaptureConnection
    ) {
        
        print(#function)
    }
    func captureOutput(_ output: AVCaptureOutput, didDrop sampleBuffer: CMSampleBuffer, from connection: AVCaptureConnection) {
    
        print(#function)
    }
}
```

We need to update the session configuration to send frames through. Add the following right before the call to `session.commitConfiguration()`.

```swift
// ...

session.addOutput(framesOut)
framesOut.setSampleBufferDelegate(
    framesDelegate,
    queue: framesQueue
)

// ...
```

You will see `captureOutput(_:didOutput:from:)` calls if you run the app now.

## Handle Frames

We will begin by importing modules. We will need MetalKit, CoreImage, and CoreImage built-in filters

Add the following to the top of the file.

```swift
import MetalKit
import CoreImage
import CoreImage.CIFilterBuiltins
```

Next, we will need to create a custom `MTKView` to act as our new viewfinder.

```swift
class LegacyMetalViewfinder: MTKView {

    lazy var commandQueue = device!.makeCommandQueue()!
    lazy var context = CIContext(
        mtlCommandQueue: commandQueue,
        options: [
            .name : "FilterContext",
            .cacheIntermediates: false
        ]
    )
    lazy var renderDestination = CIRenderDestination(
        width: Int(drawableSize.width),
        height: Int(drawableSize.height),
        pixelFormat: colorPixelFormat,
        commandBuffer: nil,
        mtlTextureProvider: { self.currentDrawable!.texture }
    )
    var image: CIImage?
}

```

In order to take maximum advantage of the GPU we will set up a custom `MTKViewDelegate`. This is the object in charge of taking the images that were set on the viewfinder and rendering them on screen with the GPU.

```swift

class LegacyMetalViewFinderDelegate: NSObject, MTKViewDelegate {
    
    func mtkView(_ view: MTKView, drawableSizeWillChange size: CGSize) {
        // TODO
    }
        
    func draw(in view: MTKView) {
        guard
            let view = view as? LegacyMetalViewfinder,
            let image = view.image,
            let drawable = view.currentDrawable,
            let buffer = view.commandQueue.makeCommandBuffer()
        else { return }
        try! view.context.startTask(
            toRender: image,
            to: view.renderDestination
        )
        buffer.present(drawable)
        buffer.commit()
    }
}
```

You can watch
[Optimize the Core Image pipeline for your video app
](https://developer.apple.com/wwdc20/10008). It's a short WWDC session that explains how to get the best performance in this situation and it explains why this view was configured this way. 

Now for the SwiftUI part. Aside from minor differences, This part will be similar to setting up the plain view finder from the previous article.

```swift
struct MetalViewfinder: UIViewRepresentable {
    
    typealias UIViewType = UIView
    
    var legacyViewfinder: LegacyMetalViewfinder
    var metalDelegate = LegacyMetalViewFinderDelegate()
    
    func makeUIView(context: Context) -> UIView {
        legacyViewfinder.backgroundColor = .clear
        legacyViewfinder.delegate = metalDelegate
        return legacyViewfinder
    }
    func updateUIView(_ uiView: UIView, context: Context) {
        // TODO
    }
}
```

This object will receive an instance of `LegacyMetalViewfinder`. It will set it up with a clear background, and connect it to a `LegacyMetalViewFinderDelegate`.

## Filter Frames

Now for the exciting part, we will be updating the `FrameDelegate`. This is the object that is receiving frames from the camera.

First, we will add a property for the `LegacyMetalViewfinder` and implement an init to store and configure it for receiving frames.

```swift
class FramesDelegate:NSObject,AVCaptureVideoDataOutputSampleBufferDelegate {
    let metalViewfinder: LegacyMetalViewfinder
    
    init(viewfinder: LegacyMetalViewfinder) {
        metalViewfinder = viewfinder
        metalViewfinder.framebufferOnly = false
        
        super.init()
    }
    // ...
```

Now we can implement `captureOutput(_:didOutput:from:)`. This is where we will be taking the frames we get from the camera, modifying them with Metal optimized `CIFilters`, and sending them off to our metal based viewfinder.

```swift 
guard let imageBuffer = sampleBuffer.imageBuffer
else {
    
    // Could not get an image buffer
    return
}

// Get a CI Image
let image = CIImage(cvImageBuffer: imageBuffer)

// ...
```

Then we want to get the size of it and calculate how we want it to fit within our viewfinder.

```swift
// ...

// Get the original size of the image
let originalSize = image.extent.size
        
// A rect with our frame's aspect ratio that fits in the viewfinder
let targetRect = AVMakeRect(
    aspectRatio: originalSize,
    insideRect: CGRect(
        origin: .zero,
        size: metalViewfinder.drawableSize
    )
)

// ...
```

The `AVMakeRect` function saves us from writing geometry code to deal with ratios, orientations, and resolutions. [Check it out](https://developer.apple.com/documentation/avfoundation/1390116-avmakerect)

All we have to do now is create a couple basic transforms.

```swift
// ...

// A transform to size our frame to targetRect
let scale = CGAffineTransform(
    scaleX: targetRect.size.width / originalSize.width,
    y: targetRect.size.height / originalSize.height
)

// A transform to center our frame within the viewfinder
let translate = CGAffineTransform(
    translationX: targetRect.origin.x,
    y: targetRect.origin.y
)

// ...
```

We create a filter and use it on our `CIImage`, apply the transforms we created, and send it to the viewfinder.

```swift
// ...

// Apply a filter
let filter = CIFilter.pixellate()
filter.scale = 12

// We use clamping to resolve some issues with the pixellate filter

// TODO: Try removing the call to clamped and change the scale.
// What happens? Do you see what this does for us?
filter.inputImage = image.clampedToExtent()

// Apply the transforms to the image
let scaledImage = (filter.outputImage ?? image)
    .cropped(to: image.extent)
    .transformed(by: scale.concatenating(translate))

// Set the image on the viewfinder
metalViewfinder.image = scaledImage
```

Notice that we apply the filters before the transforms. We do this because we want the results of the filter to be consistent with the data we are getting from the sensor. For example, the image we get on an `11 Pro` should be the same as an `11 Pro Max` regardless of the difference in display size.

## Update the UI

Go to the `ContentView` and update the `framesDelegate` property

```swift
// ...

let framesDelegate = FramesDelegate(
    viewfinder: LegacyMetalViewfinder(
        frame: .zero,
        device: MTLCreateSystemDefaultDevice()!
    )
)

// ...
```

Here we are creating a metal device, using it to create a `LegacyMetalViewfinder`, and using that to now create our `FramesDelegate`.

Finally, we can update what we are returning from the body of `ContentView` to our shiny new SwiftUI + Metal Viewfinder.

```swift
// ...

return MetalViewfinder(legacyViewfinder: framesDelegate.metalViewfinder)
```

Here we have created our `MetalViewfinder` representable and passed in the same `metalViewfinder` from the `framesDelegate`.

Now you can run the app and see yourself, as a Minecraft character...

Except there is one minor issue. The frames we are getting from the camera are in the wrong orientation. We can fix this by updating the connections on the session. Add the following right before the call to `commitConfiguration`


```swift
// ...

// Connection configurations
CONNECTION : for connection in session.connections {
    
    // Handle orientation and mirroring properly
    if connection.isVideoOrientationSupported {
        connection.videoOrientation = .portrait
    }
    
    if connection.isVideoMirroringSupported {
        connection.isVideoMirrored = true
    }
}
```

Now the viewfinder is properly oriented in portrait and mirrored like a normal camera app.

[Download Project](https://github.com/ianleon/BlogCam/tree/Ep2)

Continue: [Generics, Protocol Oriented Programming, and SwiftUI](https://github.com/ianleon/Blog/blob/master/episodes/ep3.md)
