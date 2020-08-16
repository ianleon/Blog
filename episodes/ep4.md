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

In `ContentView` we can now use the viewfinder's filter property.

Add the following right before the return statement in the `body` property

```swift
// ...
let zoom = CIFilter.zoomBlur()
zoom.amount = 20
zoom.center = .init(x: 500, y: 500)

viewfinder.filter = zoom
// ...
```

We have created a different filter and set it on the viewfinder. This is a better design because now we can reuse that same filter, with the same configuration, to process captured images and save them to the photos library.

As a fun exercise, we can try various filters. Here is a set of filter configurations that we can try passing to the viewfinder. Try changing the values and see what they do.

```swift
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

## Creating a `CIFilter` Pipeline

Using a single filter is fun. However, `CoreImage` is an optimized system. We can push it further. 

You may want to checkout the documentation for `CoreImage`: 

- [`CoreImage`](https://developer.apple.com/documentation/coreimage)
- [`CoreImage` - `CIImage`](https://developer.apple.com/documentation/coreimage/ciimage)

`CoreImage` was built to chain filters together.

However, this requires getting the `outputImage` from a filter and setting it as the `inputImage` for the following filter in the chain.

We want to avoid that verbose manual chaining. 

We do not want to change the viewfinder code. Instead we want to write the following code in our `ContentView`.

```swift
viewfinder.filter = .pipeline([pixellate, zoom])
```

With Swift's extensible and Protocol Oriented Programming features, that code can work. What's even more amazing is that we wont change a single character in the current viewfinder implementation 🤯.

We will create a new `FilterPipeline` type. 

```swift
class FilterPipeline: CIFilter {
    
    // The filters in the order they will be applied
    var filterChain: [CIFilter]
    
    // Allow setting the input image through key value coding
    @objc var inputImage: CIImage?
    
    init(_ filters: [CIFilter]) {
        filterChain = filters
        super.init()
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    override var outputImage: CIImage? {
        get {

            // Run the `inputImage` through the `filterChain`
            filterChain.reduce(inputImage) {
                image, filter -> CIImage? in
                
                // Apply the current `filter` to the `image`
                filter.setValue(image, forKey: kCIInputImageKey)
                
                // Return the new image
                return filter.outputImage
            }
        }
    }
}

extension CIFilter {
    static func pipeline(_ filters: [CIFilter]) -> FilterPipeline {
        return FilterPipeline(filters)
    }
}
```

It is a subclass of `CIFilter`. This is what allows us to substitute it for the standard single filters. We have added an `inputImage` property with an `@objc` attribute to make it key value coding compliant like the standard filters. We implement the `init` methods. For the `outputImage` property we want to apply all the filters in `filterChain` to our `inputImage`. The extension on `CIFilter` allows us to have a similar instantiation syntax as the built in filters.

Now we are able to chain multiple filters together in a clear way. Since these pipelines are substitutable for filters you can nest pipelines within pipelines.

```swift
viewfinder.filter = .pipeline([pixellate, .pipeline([zoom,hue])])
```

Being able to write code like the above will allow us to try out more filter combinations in less time.

[Download Project](https://github.com/ianleon/BlogCam/tree/Ep4)
