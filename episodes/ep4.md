# Compose Filter Pipelines with Swift - Episode 4

<!-- statement of empowerment -->
Read this article to learn how you can use Swift's extensible and Protocol Oriented Programming features to implement a CoreImage `CIFilter` pipeline.

<!-- status update -->
We will be improving upon our SwiftUI camera app by implementing a `CIFilter` pipeline for our viewfinder.

<!-- tool and environment version -->
I will be using Xcode 12 beta 4 on Big Sur Beta 4.

<!-- call to action -->
Use the project from the last episode as a starting point to code-along.

<!-- material download -->
[EP 3 Project](https://github.com/ianleon/BlogCam/tree/Ep3)

## Setup `CIFilter` outside Viewfinder

Currently we have one filter being instantiated, configured, and used directly inside of the the viewfinder's  `captureOutput(_:didOutput:from:)` delegate method. The viewfinder should have a filter property that can be changed externally. The `captureOutput(_:didOutput:from:)` should be able to use the arbitrary filter in this property to set an input image, get back an output image, and render it to the screen.

First we add an optional filter property to the viewfinder.

```swift
class Viewfinder: MTKView {

    /// Filter applied to `image`
    var filter: CIFilter?
    
    // ...
}
```


The we update the `captureOutput(_:didOutput:from:)` implementation to use this filter property instead of the locally created filter.

```swift
// ... 

// Before
let filter = CIFilter.pixellate()
filter.scale = 12
filter.inputImage = sampleImage.clampedToExtent()

// After
filter?.setValue(
    sampleImage.clampedToExtent(),
    forKey: kCIInputImageKey
)

// ...
```

Notice that we have to set the input image property through key-value coding. This is because `CIFilter` does not have an `inputImage` property. Instead, the specific filters that we get through `CIFilter.filterName()` have their own `inputImage` properties.

In `ContentView` we can use the filter property.

Add the following right before the return statement in the `body` property

```swift
// ...
let zoom = CIFilter.zoomBlur()
zoom.amount = 20
zoom.center = .init(x: 500, y: 500)

viewfinder.filter = zoom
// ...
```

We have created a different filter from the one that we had before and set it on the viewfinder. This is a better design because now we can reuse that same filter, with the same configuration, to process captured images and save them to the photos library.

As a fun exercise, we can try various filters. Here is a set of filter configurations that we can try passing to the viewfinder. Try changing the values and see what they do.

```
let pixellate = CIFilter.pixellate()
pixellate.scale = 12

let invert = CIFilter.colorInvert()

let hole = CIFilter.holeDistortion()

let hue = CIFilter.hueAdjust()
hue.angle = .pi

let sharp = CIFilter.sharpenLuminance()
sharp.sharpness = 40

let unsharp = CIFilter.unsharpMask()
unsharp.radius = 40
        
let bloom = CIFilter.bloom()
bloom.radius = 25

let convolution = CIFilter.convolution3X3()
convolution.weights = .init(values: [  0, 1,0,
                                      -1, 0,1,
                                       0,-1,0], count: 9)
convolution.bias = 0.6

let motionBlur = CIFilter.motionBlur()
motionBlur.radius = 20

let zoom = CIFilter.zoomBlur()
zoom.amount = 20
zoom.center = .init(x: 500, y: 500)
```

## 

[Download Project](https://github.com/ianleon/BlogCam/tree/Ep4)
