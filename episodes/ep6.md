# Capturing & Saving an Image - Episode 6

<!-- statement of empowerment -->
Read this article to learn how you can use iOS's frameworks to capture and save images to the Photos App.

<!-- status update -->

<!-- tool and environment version -->
I will be using Xcode 13.4.1 on Monterrey

<!-- call to action -->
Use the project from the last episode as a starting point to code-along.

<!-- material download -->
[EP 5 Project](https://github.com/ianleon/BlogCam/tree/Ep5)

## Capturing an Image

## Saving an Image

### Asking for permission to save an image

To ask the user for permission to save to their photo library we need to write a usage description.

Add the following to the `Info.plist`.

```plist
<key>NSPhotoLibraryUsageDescription</key>
<string>Saving to your photo library</string>
```
