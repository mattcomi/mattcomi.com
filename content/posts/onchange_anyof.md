+++
date = '2025-10-24T14:47:21+08:00'
draft = false
title = 'onChange(anyOf:initial:_:)'
+++

In SwiftUI, the `onChange(of:initial:_:)` modifier performs an action when the given value changes. A limitation of this modifier is that it only accepts a single value. If you want to perform _the same_ action in response to multiple changes, you’ll usually need multiple modifiers.

```swift
.onChange(of: firstName) {
  saveUser()
}
.onChange(of: lastName) {
  saveUser()
}
```

# The Workaround

If the values are of the same type, you can put them in an array:

```swift
// compiles
.onChange(of: [firstName, lastName]) {}
```

This works because [`Array` is an equatable container](https://www.swift.org/blog/conditional-conformance/#equatable-containers);  an `Array` conforms to `Equatable` if its `Element` is `Equatable`. But that doesn't work for heterogeneous collections:

```swift
// doesn't compile
.onChange(of: [firstName, lastName, age]) {}
```

You may try reaching for a tuple:

```swift
// how about…
.onChange(of: (firstName, lastName, age)) {}
```

But tuples can't conform to protocols. In other words, a tuple cannot be `Equatable` and `onChange` requires an `Equatable`. You may have noticed that this works:

```swift
(1, "A") == (1, "A") // true
```

But that's only because the language includes overloads of the `==` operator for tuples of up to [six elements](https://developer.apple.com/documentation/swift/==(_:_:)-1ud2a).

You know what can conform to a protocol though? A variadic generic.

# The Solution

Here is a solution that leverages variadic generics, and piggybacks on the existing `onChange(of:initial:_:)` modifier. First, the generic:

```swift
struct Equatables<each T>: Equatable where repeat each T: Equatable {
  static func ==(
    lhs: Equatables<repeat each T>, rhs: Equatables<repeat each T>
  ) -> Bool {
    var result = true
    repeat (result = result && (each lhs.values == each rhs.values))
    return result
  }

  var values: (repeat each T)
}
```

This type conforms to `Equatable` and declares a [type parameter pack](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0393-parameter-packs.md#type-parameter-packs) `T` where `T` is `Equatable`. Two `Equatables` are equal when all of their parameters are equal. So, functionally, **it is an `Equatable` tuple.**

And here is how `onChange(anyOf:initial:_:)` is implemented:

```swift
extension View {
  func onChange<each T>(
    anyOf values: repeat each T,
    initial: Bool = false,
    _ action: @escaping (Equatables<repeat each T>, Equatables<repeat each T>) -> Void
  ) -> some View where repeat each T: Equatable {
    self.onChange(
      of: Equatables<repeat each T>(values: (repeat each values)),
      initial: initial
    ) { oldValue, newValue in
      action(oldValue, newValue)
    }
  }

  func onChange<each T>(
    anyOf values: repeat each T,
    initial: Bool = false,
    _ action: @escaping () -> Void
  ) -> some View where repeat each T: Equatable {
      onChange(
        of: Equatables<repeat each T>(values: (repeat each values)),
        initial: initial,
        action
      )
  }
}
```

There are two overloads just as there are two overloads of `onChange(of:initial:_:)`. The modifiers let you do this:

```swift
struct ContentView: View {
  @State var number = 0
  @State var string = "Apple"

  var body: some View {
    VStack {
      Button {
        string = "Banana"
      } label: {
        Text("A")
      }
    }
    .onChange(anyOf: number, string, initial: true) {
      print($0.values.1) // "Apple"
      print($1.values.1) // "Banana"
    }
  }
}
```

# A Caveat

If it were possible, I'd improve the ergonomics here. You'll notice that the parameters to the action closure are two `Equatables`. In the example above, when the call site wants to access `string`, it has to call `$0.values.1` rather than simply `$0.1`. Unfortunately, passing `(repeat each T)` into a closure [crashes the compiler](https://hachyderm.io/@mattcomi/115433541894504137). Here is a reproducible example:

```swift
struct Foo<each T> {
  var values: (repeat each T)
}

func bar<each T>(
  foo: Foo<repeat each T>, 
  action: ((repeat each T)) -> Void
) {
  action(foo.values)
}
```

So until that gets resolved, we'll just have to settle for typing `.values`.