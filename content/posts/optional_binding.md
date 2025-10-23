+++
date = '2025-10-22T22:18:45+08:00'
draft = false
title = 'Don’t Unwrap Optional Bindings'
summary = 'Binding has an initializer that turns a Binding<Value?> into a Binding?<Value>. You probably don’t want this.'
+++

SwiftUI's `Binding` has an initializer that turns a `Binding<Value?>` into a `Binding?<Value>`. This means that if you have an optional `@State` property, like this:

```swift
@State var user: User?
```

You can initialize a `Binding` to it like this:

```swift
if let user = Binding($user) {
}
```

That makes it possible to do this:

```swift
if let user = Binding($user) {
  ProfileView(user: user)
}
```

Possible, but *unsafe*. If `user` becomes `nil`, your app will crash. 

Here is a simple example:

```swift
struct OuterView: View {
  @State var number: Int? = 1
 
  var body: some View {
    if let binding = Binding($number) {
      InnerView(number: binding)
    }
 
    Button("Crash") { number = nil }
  }
}
 
struct InnerView: View {
  @Binding var number: Int
 
  var body: some View {
    Text(number.formatted())
  }
}
```

The crash occurs here:

```swift
BindingOperations.ForceUnwrapping.get(base:) + 160
```

Somehow, `InnerView` gets evaluated when `number` is `nil` - even though `InnerView` can't exist in that state.

# The Alternative

My preferred workaround involves adding an extension to `Optional`:
  
```swift
extension Optional {
  subscript(default default: Wrapped) -> Wrapped {
    get {
      self ?? `default`
    }
    set {
      self = newValue
    }
  }
}
```

This subscript effectively turns an `Optional<Wrapped>` into a `Wrapped`. If the wrapped value is nil, it returns the provided `default`. 
  
Use it like this:

```swift
if number != nil {
  InnerView(number: $number[default: 0])
}
```

You will never *see* this default, but it will be required as the view hierarchy is re-evaluated. I've posted this finding on the [Apple Developer Forums](https://developer.apple.com/forums/thread/775817). If you have any insight as to *why* this is happening, please [let me know](https://mastodon.social/@mattcomi)!