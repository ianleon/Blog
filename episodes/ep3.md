# Generics, Protocol Oriented Design, and SwiftUI - Episode 3

We will be continuing our SwiftUI camera app by refactoring and making better use of Swift's features.

I will be using Xcode 12 beta 4 on Big Sur Beta 4.

Use the project from the last episode as a starting point to code-along.

[EP 2 Project](https://github.com/ianleon/BlogCam/tree/Ep2)

## Basic Housekeeping

We will delete old unused types in our app.

Delete `LegacyViewfinder` and `Viewfinder`. These were the implementation for the plain viewfinder without filters.

Delete `ContentView_Previews` since this type of app doesn't work with previews at the current time.

## Object Oriented Design

As it is, the code that makes the viewfinder work is spread across several types. In hindsight, this doesn't do much for us. For example, the `FramesDelegate` type is tightly coupled with `LegacyMetalViewfinder`. 

The way I see it there are two ways to go. The first would be to make these objects interact with each other through abstract interfaces instead of concrete types. The second would be to combine them and make extensions that add conformance to the delegate protocols that we are using. I have opted for the latter since I don't foresee these individual types being reused elsewhere in the app. The result of this decision will allow us to have a cleaner implementation in our `ContentView`.

### Getting rid of `FramesDelegate`

Replace the `FramesDelegate` declaration with the following extension declaration. 

```swift

// Before
class FramesDelegate:NSObject,AVCaptureVideoDataOutputSampleBufferDelegate {

// After
extension LegacyMetalViewfinder: AVCaptureVideoDataOutputSampleBufferDelegate {

```

We can remove the `init` function and the `metalViewfinder` property.

Now that the viewfinder itself is our frame delegate, we need to update the implementation of `captureOutput(_:didOutput:from:)`.

`LegacyMetalViewfinder` has an `image` property. I renamed the `image` variable in `captureOutput(_:didOutput:from:)` to `sampleImage` which avoids confusion while making the code more expressive. Then I updated the rest of the function accordingly.

```swift
// ...

func captureOutput(
        _ output: AVCaptureOutput,
        didOutput sampleBuffer: CMSampleBuffer,
        from connection: AVCaptureConnection
    ) {
        
    guard let imageBuffer = sampleBuffer.imageBuffer
    else {
        
        // Could not get an image buffer
        return
    }
    
    // Get a CI Image
    let sampleImage = CIImage(cvImageBuffer: imageBuffer)

    // Get the original size of the image
    let originalSize = sampleImage.extent.size
    
    // A rect with our frame's aspect ratio that fits in the viewfinder
    let targetRect = AVMakeRect(
        aspectRatio: originalSize,
        insideRect: CGRect(
            origin: .zero,
            size: drawableSize
        )
    )
    
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
    
    // Apply a filter
    let filter = CIFilter.pixellate()
    filter.scale = 12
    filter.inputImage = sampleImage.clampedToExtent()
    
    // Apply the transforms to the image
    let scaledImage = (filter.outputImage ?? sampleImage)
        .cropped(to: sampleImage.extent)
        .transformed(by: scale.concatenating(translate))

    // Set the image on the viewfinder
    image = scaledImage
}

// ...
```

We need to add inits to `LegacyMetalViewfinder`. Here we can consolidate configurations that were done on the view from the other objects. 

```swift

override init(frame frameRect: CGRect, device: MTLDevice?) {
    super.init(frame: frameRect, device: device)
    framebufferOnly = false
    backgroundColor = .clear
    delegate = self
}

required init(coder: NSCoder) {
    fatalError("init(coder:) has not been implemented")
}

```


### Getting rid of `LegacyMetalViewFinderDelegate`

As shown above, we set `LegacyMetalViewfinder` to be its own Metal delegate. This means we can now remove `LegacyMetalViewFinderDelegate`. This will be as simple as replacing its class declaration with the following extension declaration. 

```swift
// Before
class LegacyMetalViewFinderDelegate: NSObject, MTKViewDelegate {

// After
extension LegacyMetalViewfinder: MTKViewDelegate {
```

The SwiftUI integration can be simplified now that `LegacyMetalViewfinder` is its own Metal delegate. We can get rid of the `metalDelegate` property on it and simplify the `makeUIView(context:)` method now that those properties are being set in the `init` of `LegacyMetalViewfinder`.

```swift
struct MetalViewfinder: UIViewRepresentable {
    
    typealias UIViewType = UIView
    
    var legacyViewfinder: LegacyMetalViewfinder
    
    func makeUIView(context: Context) -> UIView {
        return legacyViewfinder
    }
    func updateUIView(_ uiView: UIView, context: Context) {
        // TODO
    }
}
```

## Update ContentView

Here we will see how the changes we have made affect how our viewfinder is used.

Remove the `framesDelegate` property. We can now replace it with a `metalViewfinder` property.

```swift
// ...

// Before
let framesDelegate = FramesDelegate(
    viewfinder: LegacyMetalViewfinder(
        frame: .zero,
        device: MTLCreateSystemDefaultDevice()!
    )
)

// After
let metalViewfinder = LegacyMetalViewfinder(
    frame: .zero,
    device: MTLCreateSystemDefaultDevice()!
)

// ...
```

Instantiating a viewfinder that can be used to capture frames is now simple.

Moving on to the `body` implementation, we need to change the sample buffer delegate on our frames output object. 

```swift
// ... 

// Before
framesOut.setSampleBufferDelegate(
    framesDelegate,
    queue: framesQueue
)

// After
framesOut.setSampleBufferDelegate(
    metalViewfinder,
    queue: framesQueue
)

// ... 
```

This change makes the code expressive. It makes sense that the viewfinder would be our delegate for sample frame buffers.

Update the return statement.

```swift
// ...

// Before
return MetalViewfinder(legacyViewfinder: framesDelegate.metalViewfinder)

// After
return MetalViewfinder(legacyViewfinder: metalViewfinder)
```

The above change is the **main reason for this episode**. It adds clarity. Why would we get a reference to our viewfinder from a delegate? Now we don't have to.

## SwiftUI and generics

The above is shaping up to be a cleaner design.

However, there is one thing we can improve. `MetalViewfinder` can now be generalized to take any `UIView` and show it. We can do this by making use of generics in Swift. I have renamed it from `MetalViewfinder` to `Rep`.

I have a feeling the following will get plenty of re-use.

```swift

/// Simply shows a UIView
struct Rep<T: UIView>: UIViewRepresentable {
    
    typealias UIViewType = UIView
    var view: T
    func makeUIView(context: Context) -> UIView { view }
    func updateUIView(_ uiView: UIView, context: Context) { }
}

```

Now there is no need for the 'LegacyMetal' prefix on `LegacyMetalViewfinder`. For simplicity I have renamed it to `Viewfinder`. The `metalViewfinder` variables have been renamed to `viewfinder`.

We can change the return of the `body` of `ContentView`.

```swift
// ... 

// Before (from ep2)
return MetalViewfinder(legacyViewfinder: framesDelegate.metalViewfinder)

// After
return Rep(view: viewfinder)
```

The change above is a testament to a cleaner and more expressive design.

[Download Project](https://github.com/ianleon/BlogCam/tree/Ep3)


