# Create a Shutter Button - Episode 5

<!-- statement of empowerment -->
Read this article to learn how you create a custom button in SwiftUI.

<!-- status update -->
We will be cleaning up our ContentView by extracting some setup methods (configureSession, `makeFilter`)

<!-- tool and environment version -->
I will be using Xcode 13.4.1 on Monterrey

<!-- call to action -->
Use the project from the last episode as a starting point to code-along.

<!-- material download -->
[EP 4 Project](https://github.com/ianleon/BlogCam/tree/Ep4)

## Refactoring (configureSession, makeFilter)

The `body` of `ContentView` is getting long. 

This property should be focused on creating the view. 

We will extract `configureSession` and `makeFilter` as separate methods.

```swift
 var body: some View {
    configureSession()
    
    viewfinder.filter = makeFilter()

    return Rep(view: viewfinder)
}
```


## Creating a shutter button

We would like to add a shutter button right below the viewfinder.

In SwiftUI we use `VStack` for vertical stacking.

```swift
return VStack {
    Rep(view: viewfinder)
    Button("Shutter") { /* take a picture */ }
}
```

This gives us a text based button. We would like our shutter button to use an icon.

### Creating a custom SwiftUI View

Creating a Button that uses SFSymbols in Swift UI is simple. 

The syntax is wonderful:

```swift
Button {
    takePicture()
} label: {
    Image(systemName: "circle.fill")
}
```

You can use modifiers to increase the size of the icon.

```
Button {
    takePicture()
} label: {
    Image(systemName: "circle.fill")
        .font(
            .system(size: 36)
        )
}

```

However, as is, this button is not accessible to everyone.

You can fix that using an accessibility modifier.

```
Button {
    takePicture()
} label: {
    Image(systemName: "circle.fill")
        .font(
            .system(size: 36)
        )
}
.accessibilityLabel("Shutter")
```

By this point you might feel there is too much nesting and modifying. 

You might like a reusable way to make icon based buttons while preserving the accessibility, readability, and simplicity of a text based button:

```swift
Icon(label: "Shutter", "circle.fill") {
    takePicture()
}
```

This interface is designed with the following principles in mind:

- **Readable** 

	It’s clear that this a representation of a Camera’s shutter button in code because it is labeled “Shutter”.

- **Familiar**
  
	This code looks similar to the standard SwiftUI `Button`. The custom views should harmonize with the standard views.
  
- **Accessible**
	
	Enables "Tap `Shutter`" command through *VoiceControl*
	 
- **Clean**

	Nesting is used solely for the action closure.

With SwiftUI you can implement that interface:

1. Start with an Icon struct that implements View

	```swift
	import SwiftUI
	struct Icon: View {
		let systemImageName: String
		let accessibilityLabel: String
		let action: () -> Void
	}
	```

2. Create a custom init.

	Swift automatically generates inits for structs. 
	
	However, `accessibilityLabel` is a long argument name.
	
	You can declare your own init to shorten, remove, and/or provide defaults for these arguments.
	
	```swift
	init(
		label accessibilityLabel: String,
		_ systemImageName: String = "a",
		action: @escaping () -> Void = { }
	) {
		self.systemImageName = systemImageName
		self.accessibilityLabel = accessibilityLabel
		self.action = action
	}
	```
	
	- This init shortens the `accessibilityLabel` argument to `label`. Unlike the next 2 arguments, it does not set a default value by using `=`. 
		
		This forces the caller to set a label for their `Icon`. Making the code readable and keeping the Button **accessible**.
	
	- The `_` before `systemImageName` allows you to drop that argument name at the call-site. 
	
	- Keeping the action argument last enables the trailing closure syntax. (Like SwiftUI's standard button)
	
3. Create body

	One of the great things about SwiftUI is the system will call `body` exclusively to create your view. 
	
	There is no other entry-point. This avoids a whole class of bugs. 
	
	However, you can *split* your `body` into smaller pieces by extracting those pieces into methods.
	
	This keeps your code feeling readable, clean, and lightweight.
	
	```swift
	fileprivate func icon() -> some View {
	  Image(systemName: systemImageName)
	    .font(.system(size: 36))
	}
    
	var body: some View {
	  Button(action: action, label: icon)
	    .accessibilityLabel(accessibilityLabel)
	}
	```
	
	[Download Project](https://github.com/ianleon/BlogCam/tree/Ep5)

