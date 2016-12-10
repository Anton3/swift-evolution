# Generic protocols

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Anton Zhilin](https://github.com/Anton3), [Brent Royal-Gordon](https://github.com/brentdax)
* Status: **Awaiting review**
* Review manager: TBD

## Introduction

In addition to protocols with associated types, allow generic protocols. Example:

```swift
protocol From<T> {
  init(_ from: T)
}
```

Swift-evolution thread: [Discussion thread topic for that proposal](http://news.gmane.org/gmane.comp.lang.swift.evolution)

## Motivation

It is currently not allowed to conform to multiple instances of the same protocol.
One of the reasons behind that is that even with sufficient syntax, there would be
conflicting instances of same `associatedtype` requirements:

```swift
protocol From {
  associatedtype Value
  init(_ from: Value)
}

// multiple conformances are not allowed in Swift 2.2
extension MyType : From {
  typealias Value = Int
}
extension MyType : From {
  typealias Value = Double
}

MyType.Value  // what should that even be?
```

### Generic manifesto

In generic manifesto, generic protocols are marked as "unlikely". See **[Brent's counterarguments](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160606/020746.html)**.

## Proposed solution

Allow generic protocols.
Usage of such protocol without generic parameter list is not allowed.
Multiple instantiations of the same generic protocol are treated as unrelated protocols.

### Example

```swift
protocol From<T> {
  init(_ from: T)
}

extension Int : From<Double> { }
extension Int : From<Int8> { }

func convert<T: From<Double>>(num: Double) -> T {
  return T(num)
}
print(convert(3.14) as Int)  //=> 3
```

## Detailed design

These requirements will usually not introduce any special cases in Swift compiler implementation.
They follow from "instantiation of generic protocol becomes a rightful protocol".

### Conflicting `associatedtype` requirements

To conform to multiple instantiations of the same generic protocol,
there must not be any conflicts in conforming type. In particular,
if the same `associatedtype` requirement becomes different for different conformances, we have a conflict.

```swift
protocol Converter<Input> {
  associatedtype Result
  func convert(x: Input) -> Result
}

struct GoodConverter {
  func convert(x: Int) -> String
  func convert(x: Double) -> String
}
extension GoodConverter : Converter<Int> {
  typealias Result = String  // or it can be inferred
}
extension GoodConverter : Converter<Double> {
  // typealias Result = String  // declarations must not be duplicated
}

struct BadConverter {
  func convert(x: String) -> Int
  func convert(x: String) -> Double
}
extension BadConverter : Converter<Int> {
  typealias Result = Int  // cannot be inferred
}
extension BadConverter : Converter<Double> {
  typealias Result = Double  // error: conflicting associatedtype!
}
```

In this example, there is no way `BadConverter` can conform to both `Converter<Int>` and `Converter<Double>`,
because it means conflict in `associatedtype Result`.

### Pseudo-conflicting requirements

If generic type contains requirements that do not use generic parameter, they will be "merged" in conforming type:

```swift
protocol MyProtocol<T> {
  func nonGeneric(num: Int) -> Int
  func generic(left: T, right: T) -> T
}

struct MyStruct : MyProtocol<Int>, MyProtocol<Double> {
  func nonGeneric(num: Int) -> Int
  func generic(left: Int, right: Int) -> Int
  func generic(left: Double, right: Double) -> Double
}
```

We need to implement only one `nonGeneric` function to conform to both instantiations. Such functions must not be duplicated.

### Clarification on "old" protocols

With this proposal accepted, we will be able to define *new* protocols, to which we can conform multiple times.
Old `associatedtype`-based protocols will not gain this ability.
It will still be an error to conform to `SequenceType` multiple times with different `associatedtype`s.

### Syntax in protocol extensions

```swift
protocol MyComparable<T> {
  func < (left: Self, right: T)
}
extension MyComparable {
  func > (left: T, right: Self) { ... }
}
extension MyComparable where T: Equatable {
  ...
}
```

### Existential generic protocols

If a generic protocol does not use `Self` and has no `associatedtype` requirements,
then its instantiations can be used as existentials to line up with non-generic protocols:

```swift
protocol Into<T> {
  func into() -> T
}
func convert(_ obj: Into<Double>) -> Double {
  return obj.into()
}
```

## Impact on existing code

This is a strictly additive feature.

## Alternatives considered

### Allow pseudo-generic syntax in extensions

```swift
protocol From {
  associatedtype Value
  init(_ from: Value)
}

extension Int : From where Value = Double { }
extension Int : From where Value = Int8 { }

Int.Value  // still an ambiguity :(

func convert1<T: From where T.Value == Double>(num: Double) -> T
convert1(3.14) as Int

func convert2<T: From>(num: T.Value) -> T
convert1(3.14) as Int  // still an ambiguity :(
```

## Future directions

### Protocol for usage in `as?` and `as!`

```swift
protocol Upcastable<Supertype> {
	init?(attemptingCastFrom value: Supertype)
	func casting() -> Supertype
}
```

The following axiom must hold for conforming types:

```swift
For all types Subtype: Upcastable<Supertype>
For all variables x: Subtype
Subtype.init(attemptingCastFrom: x.casting() as Supertype)  ==  .some(x)
```

With this protocol, `x as? T` should use `T(attemptingCastFrom: x)` if corresponding `Upcastable` conformance exists.
Likewise, `x as! T` should use `T(attemptingCastFrom: x)!`.

### Protocol for standard type conversions

Exact name is discussable, I'll follow Rust here.

```swift
protocol From<T> {
  init(_ from: T)
}
```

This protocol can eliminate all standard Convertible-family protocols.

It can also be used by `try` to automatically cast error types:

```swift
enum Error1 : ErrorType { ... }
enum Error2 : ErrorType, From<Error1> { ... }

func internal() -> Error1 { ... }

func canThrow() -> Error2 {
  ...
  try internal()  // automatically uses From<Error1>
  ...
}
```
