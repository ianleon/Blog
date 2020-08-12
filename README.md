# Using SwiftUI to make a camera app

I've always wanted to make a camera app. 

I always suspected that the iPhone hardware and the building blocks that Apple gives us in their API had a multitude of options and settings that common camera apps do not expose. My goal is to create a camera app that exposes and exercises everything in the hardware and in the API without too much regard for user experience or user interface.

To make this challenging, I have decided to build this app in SwiftUI. There aren't any SwiftUI components for AVKit. Therefore, I will have to use `Framework Integration` with `UIKit`.

This app will become my testing ground / laboratory for exploring other image related technology in iOS. For example, I will be using Core Image, Vision, Metal, ARKit, and anything else that might come along.

This will not be a user friendly application. For example, if you refuse to give this app permission to use the camera, it will crash. This app will not helpfully guide you to settings to nudge you to enable camera access. This app is not intended to go on the App Store.

[▶️ Start Watching](/episodes/ep1.md)
