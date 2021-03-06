# Extend Property Wrappers to Function and Closure Parameters

* Proposal: [SE-0293](0293-extend-property-wrappers-to-function-and-closure-parameters.md)
* Authors: [Holly Borla](https://github.com/hborla), [Filip Sakel](https://github.com/filip-sakel)
* Review Manager: [Chris Lattner](https://github.com/lattner)
* Status: **Scheduled for review (January 27...February 8, 2021)**
* Implementation: [apple/swift#34272](https://github.com/apple/swift/pull/34272)
* Decision Notes: [Review #1](https://forums.swift.org/t/returned-for-revision-se-0293-extend-property-wrappers-to-function-and-closure-parameters/42953)
* Previous versions: [Revision #1](https://github.com/apple/swift-evolution/blob/e5b2ce1fd6c1c2617a820a1e6f2b53a00e54fdce/proposals/0293-extend-property-wrappers-to-function-and-closure-parameters.md)

## Contents

+ [Introduction](#introduction)
+ [Motivation](#motivation)
  - [Argument validation](#argument-validation)
  - [Pass-by-value for reference types](#pass-by-value-for-reference-types)
  - [Memberwise initialization](#memberwise-initialization)
+ [Proposed solution](#proposed-solution)
+ [Detailed design](#detailed-design)
  - [Function-body semantics](#function-body-semantics)
  - [Call-site semantics](#call-site-semantics)
    - [Passing a projected value argument](#passing-a-projected-value-argument)
    - [Arguments in the property-wrapper attribute](#arguments-in-the-property-wrapper-attribute)
    - [Overload resolution of backing property-wrapper initializer](#overload-resolution-of-backing-property-wrapper-initializer)
  - [Semantics of function expressions](#semantics-of-function-expressions)
    - [Unapplied function references](#unapplied-function-references)
    - [Closures](#closures)
  - [Restrictions on property-wrapper parameters](#restrictions-on-property-wrapper-parameters)
+ [Source compatibility](#source-compatibility)
+ [Effect on ABI stability](#effect-on-abi-stability)
+ [Effect on API resilience](#effect-on-api-resilience)
+ [Alternatives considered](#alternatives-considered)
  - [Callee-side property wrapper application](#callee-side-property-wrapper-application)
  - [Passing a property-wrapper storage instance directly](#passing-a-property-wrapper-storage-instance-directly)
+ [Future directions](#future-directions)
  - [Generalized property-wrapper initialization from a projection](#generalized-property-wrapper-initialization-from-a-projection)
  - [Extending property wrappers to patterns](#extending-property-wrappers-to-patterns)
  - [Property-wrapper parameters in memberwise initializers](#property-wrapper-parameters-in-memberwise-initializers)
  - [Support `inout` in wrapped function parameters](#support-`inout`-in-wrapped-function-parameters)
  - [Wrapper types in the standard library](#wrapper-types-in-the-standard-library)
+ [Revisions](#revisions)
  - [Changes from the first reviewed version](#changes-from-the-first-reviewed-version)
+ [Appendix](#appendix)
  - [Mutability of composed `wrappedValue` accessors](#mutability-of-composed-wrappedValue-accessors)
+ [Acknowledgements](#acknowledgements)


## Introduction

Property Wrappers were [introduced in Swift 5.1](https://github.com/apple/swift-evolution/blob/main/proposals/0258-property-wrappers.md), and have since become a popular mechanism for abstracting away common accessor patterns for properties. Currently, applying a property wrapper is solely permitted on local variables and type properties. However, with increasing adoption, demand for extending _where_ property wrappers can be applied has emerged. This proposal aims to extend property wrappers to function and closure parameters.


## Motivation

Property wrappers have undoubtably been very successful. Applying a property wrapper to a property is enabled by an incredibly lightweight and expressive syntax. For instance, frameworks such as [SwiftUI](https://developer.apple.com/documentation/swiftui/) and [Combine](https://developer.apple.com/documentation/combine) introduce property wrappers such as [`State`](https://developer.apple.com/documentation/swiftui/state), [`Binding`](https://developer.apple.com/documentation/swiftui/binding) and [`Published`](https://developer.apple.com/documentation/combine/published) to expose elaborate behavior through a succinct interface, helping craft expressive yet simple APIs. However, property wrappers are only applicable to local variables and type properties, shattering the illusion that they helped realize in the first place when working with parameters.

Property wrappers attached to parameters have a wide variety of use cases. We present a few examples here.

### Argument validation

Both library developers and language users often need to assert their assumptions, for which `precondition(_:_:)` is often used:

```swift
enum Product {
  case plainSandwich 
  case grilledCheeseSandwich 
  case avocadoToast
}

func buy(quantity: Int, of product: Product) {
  precondition(quanity >= 1, "Invalid product quanity.")
  
  if quantity == 1 {
    ...
  }
}
```

The above code is quite clear; it has, though, the obvious drawback that changing the the condition to be asserted or its error message requires significant effort as a precondition statement is individually written for each function and manually documented.

Furthermore, supposing the above is library code, we may want to test for our precondition while offering an easy-to-debug way. So, using `Validation` from [`PropertyKit`](https://github.com/SvenTiigi/ValidatedPropertyKit) we can write:

```swift
@propertyWrapper
struct Asserted<Value> {
  // The assertion will appear at the right file and line
  init(
    wrappedValue: Value, 
    validation: Validation<Value>,
    file: StaticString = #file,
    line: UInt = #line
  ) {
    ...
  }

  var wrappedValue: Value { ... }

  var projectedValue: Result<Value, ValidationResult> { ... }
}


func buy(
  quantity: Int, 
  of product: Product,
  file: StaticString = #file,
  line: UInt = #line
  // These are a lot of properties for every time
  // we want to check for the right quantity.
) {
  var validatedQuantity = Asserted(
    quantity,
    .greaterOrEqual(1),
    file: file, 
    line: line
  )
  
  if validatedQuantity.wrappedValue == 1 {
    // The lack of property-wrapper tranformation is
    // evident here.
    ...
  }
}
```

This not only makes writing easy-to-maintain validations easy, but improves debugging for the the API's users as well. Unfortunately, it still lacks the elegant syntax property wrappers offer, making the creation of new functions a demanding task.

### Pass-by-value for reference types

The `@NSCopying` attribute is a tool to emulate value semantics for reference-type properties. The same functionality can now be implemented as a property wrapper, as shown in [SE-0258](https://github.com/apple/swift-evolution/blob/master/proposals/0258-property-wrappers.md#nscopying):

```swift
@propertyWrapper
struct Copying<Value: NSCopying> {
  private var _value: Value

  init(wrappedValue value: Value) {
    // Copy the value on initialization.
    self._value = value.copy() as! Value
  }

  var wrappedValue: Value {
    get { return _value }
    set {
      // Copy the value on reassignment.
      _value = newValue.copy() as! Value
    }
  }
}
```

However, this property wrapper cannot be used on parameters to achieve pass-by-value semantics for reference-type arguments. To achieve pass-by-value semantics, `copy()` must be called manually, which is easy to forget, or the `Copying` type must be used directly in an API, which causes each call-site to manually create an instance of `Copying`.

### Memberwise initialization

Consider the following property wrapper, inspired by `@Traceable` from [David Piper's blog post](https://medium.com/better-programming/creating-a-history-with-property-wrappers-in-swift-5-1-4c0202060a7f), which tracks the history of a value:

```swift
struct History<Value> { ... }

@propertyWrapper
struct Traceable<Value> {

  init(wrappedValue value: Value) { ... }

  init(projectedValue: History<Value>) { ... }

  var wrappedValue: Value {
    get {
      return history.currentValue
    }
    set {
      history.append(newValue)
    }
  }

  var projectedValue: History<Value> { return history }

  private var history: History<Value>

}
```

This property wrapper can be initialized with a value to be traced, or with an existing history of a value being traced. Now consider the following model for a simple text editor that supports change tracking:

```swift
struct TextEditor {
  @Traceable var dataSource: String
}
```

Currently, property-wrapper attributes on struct properties interact with function parameters through the struct's synthesized memberwise initializer. Because the `@Traceable` property wrapper supports initialization from a wrapped value via `init(wrappedValue:)`, the memberwise initializer for `TextEditor` will take in a `String`. However, the programmer may want to initialize `TextEditor` with a string value that already has a history. Today, this can be achieved with overloads, which can greatly impact compile-time performance, or by exposing the `Traceable` type through the `TextEditor` initializer, which is meant to be implementation detail.

## Proposed solution

We propose to allow application of property wrappers on function and closure parameters, allowing the call-site to pass a wrapped value or a projected value which will be used to automatically initialize the backing property wrapper.

Using property-wrapper parameters, the above argument validation example can be simplified to:

```swift
func buy(
  @Asserted(.greaterOrEqual(1)) quantity: Int,
  of product: Product,
) {
  if quantity == 1 {
    ...
  }
}
```

## Detailed design

Annotating a parameter declaration with a property-wrapper attribute allows the call-site to pass a wrapped value or a projected value, and the compiler will automatically initialize the backing wrapper to be passed to the function. The function author can also use the property-wrapper syntax for accessing the backing wrapper and the projected value within the body of the function. This behavior merely builds on property wrappers’ being sugar wrapping a given declaration with compiler-synthesized code.

### Function-body semantics

Attaching a property wrapper to a parameter makes that parameter a computed variable local to the function body, and changes the parameter type to the backing wrapper type. The type of the parameter is only observable in compiled code - [unapplied references to functions with property-wrapped parameters](#unapplied-function-references) will not use the backing-wrapper type.

The transformation of function with a property-wrapped parameter will be performed as such:

1. The argument label will remain unchanged. 
2. The parameter name will be prefixed with an underscore.
3. The type of the parameter will be the backing property-wrapper type.
4. A local computed property representing the `wrappedValue` of the innermost property wrapper will be synthesized with the same name as the original, unprefixed parameter name. If the innermost `wrappedValue` defines a setter, a setter will be synthesized for the local property if the mutability of the composed setter is `nonmutating`. The mutability computation is specified in the [appendix](#appendix).
5. If the outermost property wrapper defines a `projectedValue` property with a `nonmutating` getter, a local computed property representing the outermost `projectedValue` will be synthesized and named per the original parameter name prefixed with a dollar sign (`$`). If the outermost `projectedValue` defines a setter, a setter for the local computed property will be synthesized if the `projectedValue` setter is `nonmutating`.

Consider the following function with a property-wrapped parameter using the `@Asserted` property wrapper:

```swift
func insert(@Asserted(.nonEmpty) text: String) { ... }
```

The compiler will synthesize computed `text` and `$text` variables in the body of `insert`:

```swift
func insert(text _text: Asserted<String>) {
  var text: String {
    get { _text.wrappedValue }
  }

  var $text: Result<String, ValidationResult> {
    get { _text.projectedValue }
  }

  ...
}
```

### Call-site semantics

When passing an argument to a function with a property-wrapped parameter, the compiler will wrap the argument in a call to the appropriate initializer depending on the argument label. When using the original argument label (or no argument label), the compiler will wrap the argument in a call to `init(wrappedValue:)`. When using the argument label prefixed with `$` (or `$_` in the case of no argument label), the compiler will wrap the argument in a call to `init(projectedValue:)`. 

Consider the `@Traceable` property wrapper that implements both `init(wrappedValue:)` and `init(projectedValue:)`:

```swift
struct History<Value> { ... }

@propertyWrapper
struct Traceable<Value> {

  init(wrappedValue value: Value)

  init(projectedValue: History<Value>)

  var wrappedValue: Value

  var projectedValue: History<Value>

}
```

A function with an `@Traceable` parameter can be called with either a wrapped value or a projected value:

```swift
func log<Value>(@Traceable value: Value) { ... }

let history: History<Int> = ...
log(value: 10)
log($value: history)
```

The compiler will inject a call to the appropriate property-wrapper initializer into each call to `log` based on the argument label, so the above code is transformed to:

```swift
log(value: Traceable(wrappedValue: 10))
log(value: Traceable(projectedValue: history))
```

Wrapped parameters with no argument label can still be passed a projection using the syntax `$_:`, as shown in the following example:

```swift
func log<Value>(@Traceable _ value: Value) { ... }

let history: History<Int> = ...
log(10)
log(_: 10)
log($_: history)
```

For composed property wrappers, initialization of the backing wrapper via wrapped value will contain a call to `init(wrappedValue:)` for each property-wrapper attribute in the composition chain. However, initialization via projected value will only contain one call to `init(projectedValue:)` for the outermost wrapper attribute, because property wrapper projections are not composed. For example: 

```swift
func log(@Traceable @Traceable text: String) { ... }

let history: History<Traceable<String>> = ...
log(text: "Hello!")
log($text: history)
```

The above calls to `log` are transformed to:

```swift
log(text: Traceable(wrappedValue: Traceable(wrappedValue: "Hello!"))
log(text: Traceable(projectedValue: history))
```

This transformation at the call-site only applies when calling the function directly using the declaration name. The semantics of closures and unapplied function references are specified [in a later section](#semantics-of-function-expressions).

#### Passing a projected value argument

Property-wrapper projections are designed to allow property wrappers to provide a representation of the storage type that can be used outside of the context that owns the property-wrapper storage. Typically, projections either expose the backing property wrapper directly, or provide an instance of a separate type that vends more restricted access to the functionality on the property wrapper.

When a property-wrapper has a projection, it's often the case that the presence of the property-wrapper is fundamental to understanding the behavior of the property itself, and it's often necessary to use the projection alongside the wrapped value. In such cases, the projection is equal in importance to the wrapped value in the API of the wrapped property. With respect to function parameters, it's equally important to support passing a projection.

Property wrappers can opt into passing a projected-value argument to a property-wrapped parameter. To enable passing a property-wrapper projection to a function with a wrapped parameter, property wrappers must declare `var projectedValue`, and implement an `init(projectedValue:)` that meets the following requirements:

- The first parameter of this initializer must be labeled `projectedValue` and have the same type as the `var projectedValue` property.
- The initializer must have the same access level as the property-wrapper type.
- The initializer must not be failable.
- Any additional parameters to the initializer must have default arguments.

This method of initialization is not mandatory for functions using supported wrapper types, and it can be disabled by providing arguments in the wrapper attribute, including empty attribute arguments: `func log(@Traceable() _ value: Value) { ... }`.

#### Arguments in the property-wrapper attribute

Arguments in the property-wrapper attribute as well as other default arguments to `init(wrappedValue:)` and `init(projectedValue:)` have the same semantics as default arguments. These arguments are injected into the initializer call at the call-site, and therefore are evaluated by the caller. The same restrictions applied to default arguments are also applied to property-wrapper attribute arguments.

#### Overload resolution of backing property-wrapper initializer

Since the property wrapper is initialized at the call-site, the argument type can impact overload resolution of `init(wrappedValue:)` and `init(projectedValue:)`. For example, if a property wrapper defines overloads of `init(wrappedValue:)` with different generic constraints and that wrapper is used on a function parameter, e.g.:

```swift
@propertyWrapper
struct Wrapper<Value> {

  init(wrappedValue: Value) { ... }

  init(wrappedValue: Value) where Value : Collection { ... }
  
}

func generic<T>(@Wrapper arg: T) { ... }
```

Then, overload resolution will choose which `init(wrappedValue:)` to call based on the static type of the argument at the call-site:

```swift
generic(arg: 10) // calls the unconstrained init(wrappedValue:)

generic(arg: [1, 2, 3]) // calls the constrained init(wrappedValue:)
                        // because the argument conforms to Collection.
```

### Semantics of function expressions

#### Unapplied function references

By default, unapplied references to functions that accept property-wrapped parameters use the wrapped-value type in the parameter list, and the compiler will generate a thunk to initialize the backing wrapper and call the function.

Consider the `log` function from above, which uses the `@Traceable` property wrapper:

```swift
func log<Value>(@Traceable value: Value) { ... }
```

The type of `log` is `(Value) -> Void`. These semantics can be observed when working with an unapplied reference to `log`:

```swift
let logReference: (Int) -> Void = log
logReference(10)

let logReferenceWithLabel: (Int) -> Void = log(value:)
logReferenceWithLabel(10)
```

The compiler will generate a thunk when referencing `log` to take in the wrapped-value type and initialize the backing property wrapper. Both references to `log` in the above example are transformed to:

```swift
{ log(value: Traceable(wrappedValue: $0) }
```

The type of an unapplied function reference can be changed to instead take in the projected-value type using `$` in front of the argument label. Since `Traceable` implements `init(projectedValue:)`, the `log` function can be referenced in a way that takes in `History` by using `$` in front of `value`:

```swift
let history: History<Int> = ...
let logReference: (History<Int>) -> Void = log($value:)
logReference(history)
```

If the property-wrapped parameter in `log` omitted its argument label, the function could still be referenced to take in the projected-value type using `$_`:

```swift
func log<Value>(@Traceable _ value: Value) { ... }

let history: History<Int> = ...
let logReference: (History<Int>) -> Void = log($_:)
logReference(history)
```

#### Closures

Closures have the same semantics as unapplied function references, albeit with different syntax because the property-wrapper attribute needs to be specified on the closure parameter declaration. The above `log` function can be implemented as a closure that takes in the wrapped-value type:

```swift
let log: (Int) -> Void = { (@Traceable value) in
  ...
}
```

The closure can be implemented to instead take in the projected-value type by using the `$` prefix in the parameter name:

```swift
let log: (History<Int>) -> Void { (@Traceable $value) in
  ...
}
```

For closures that take in a projected value, the property-wrapper attribute is not necessary if the backing property wrapper and the projected value have the same type, such as the [`Binding`](https://developer.apple.com/documentation/swiftui/binding) property wrapper from SwiftUI. If `Binding` implemented `init(projectedValue:)`, it could be used as a property-wrapper attribute on closure parameters without explicitly writing the attribute:

```swift
let useBinding: (Binding<Int>) -> Void = { $value in
  ...
}
```

Since property-wrapper projections are not composed, this syntax will only infer one property-wrapper attribute. To use property-wrapper composition, the attributes must always be explicitly written.

### Restrictions on property-wrapper parameters

The composed mutability of the innermost `wrappedValue` getter must be `nonmutating`.

> **Rationale**: If the composed `wrappedValue` getter is `mutating`, then the local computed property for a property-wrapper parameter must mutate the backing wrapper, which is immutable.

Property-wrapper parameters cannot have an `@autoclosure` type.

> **Rationale**: `@autoclosure` is unnecessary for the wrapped value, because the wrapped-value argument at the call-site will always be wrapped in a call to `init(wrappedValue:)` or `init(projectedValue:)`, which can _already_ support `@autoclosure` arguments.

Property-wrapper parameters cannot also have an attached result builder attribute.

> **Rationale**: Result-builder attributes can be applied to the parameters in `init(wrappedValue:)` and `init(projectedValue:)`. If there is a result builder attached to a property-wrapper parameter that already has a result builder in `init(wrappedValue:)`, it's unclear which result builder should be applied.

Property-wrapper parameters with arguments in the wrapper attribute cannot be passed a projected value.

> **Rationale** Arguments in the wrapper attribute only apply to `init(wrappedValue:)`. To ensure that these arguments never change, the call-site must always use `init(wrappedValue:)` and pass the additional attribute arguments.

Property wrappers on function parameters must support `init(wrappedValue:)`.

> **Rationale**: This is an artificial limitation to prevent programmers from writing functions with an argument label that cannot be used to call or reference the function.

Non-instance methods cannot use property wrappers that require the enclosing `self` subscript.

> **Rationale**: Non-instance methods do _not_ have an enclosing `self` instance, which is required for the local computed property that represents `wrappedValue`.

Property wrapper attributes can only be used on parameters in overridden functions or protocol witnesses if the original has the same property wrapper attributes.

> **Rationale**: This restriction ensures that the call-site transformation is always the same for families of dynamically dispatched functions.

## Source compatibility

This is an additive change with no impact on source compatibility.

## Effect on ABI stability

This is an additive change with no impact on the existing ABI.

## Effect on API resilience

This proposal makes property-wrapper custom attributes on function parameters part of public API. This is because a property wrapper applied to a function parameter changes the type of that parameter in the ABI; it also changes the way that function callers are compiled to pass an argument of that type. Thus, adding or removing a property wrapper on a public function parameter is an ABI-breaking change –– but not a source-breaking one. Furthermore, similar to the existing behavior of default arguments, arguments in wrapper attributes are emitted into clients and are therefore not a part of the ABI.

## Alternatives considered

### Callee-side property wrapper application

Instead of initializing the backing property wrapper using the argument at the call-site of a function that accepts a wrapped parameter, another approach is to initialize the backing property wrapper using the parameter in the function body. One benefit of this approach is that annotating a parameter with a property-wrapper attribute would not change the type of the function in the ABI, and therefore adding or removing a wrapper attribute would be a resilient change.

Under these semantics, using a property-wrapper parameter is effectively the same as using a local property wrapper that is initialized from a parameter. This implies that:

1. This feature cannot support passing a projected-value argument via `init(projectedValue:)`, unless there is an additional ABI entry point that acceps the projected-value type. This can only be achieved with either an exponential number of overloads, or an artificial restriction that all property-wrapper arguments must either be the wrapped-value type or the projected-value type.
2. The type of the argument provided at the call-site cannot affect the overload resolution of `init(wrappedValue:)`.
3. Arguments in the wrapper attribute and other default arguments to the property-wrapper initializers become resilient and are also evaluated in the callee rather than the caller.

One of the motivating use-cases for property-wrapper parameters is the ability to pass a projected value, which makes this approach unviable without a significant type-checking performance impact or unintuitive restrictions. Further, making arguments in the wrapper attribute resilient is inconsistent with default arguments. Finally, caller-side property wrapper application has useful semantics. For example, for property wrappers that capture the file and line number to log a message or assert a precondition, it's much more useful to capture the location where the argument is provided rather than the location of the parameter declaration.

### Passing a property-wrapper storage instance directly

A previous revision of this proposal supported passing a property-wrapper storage instance to a function with a wrapped parameter directly because the type of such a function was in terms of the property-wrapper type. A big point of criticism during the first review was that the backing storage type should be an artifact of the function implementation, and not exposed to function callers through the type system.

Exposing the property-wrapper storage type through the type system has the following implications, summarized by [Frederick Kellison-Linn](https://forums.swift.org/u/jumhyn):

* The addition/removal of a property-wrapper attribute on a function parameter is a source-breaking change for any code that references or curries the function.
* It prohibits the use of initializer arguments in the wrapper attribute. There's no point in declaring a wrapper as `@Asserted(.greaterOrEqual(1))` if any client can simply pass an `Asserted` instance with a completely different validation.
* It removes API control from both the property wrapper author and the author of the wrapped-argument function.

Keeping the property-wrapper storage type private is consistent with how property wrappers work today. Unless a property wrapper projects its storage type via `projectedValue`, the storage type itself is meant to be private, implementation detail that cannot be accessed by API clients.

## Future directions

### Generalized property-wrapper initialization from a projection

This proposal adds `init(projectedValue:)` as a new property-wrapper initialization mechanism from a projected value for function arguments. This mechanism could also be used to support definite initailization from a projected value for properties and local variables:

```swift
struct TextEditor {
  @Traceable var dataSource: String

  init(history: History<String>) {
    $dataSource = history //  treated as _dataSource = Traceable(projectedValue: history)
  }
}
```

### Extending property wrappers to patterns

Passing a property-wrapper storage instance directly to a property-wrapped closure parameter was supported in first revision. One suggestion from the core team was to imagine this functionality as an orthogonal feature to allow pattern matching to "unwrap" property wrappers. Though this proposal revised the design of closures to match the behavior of unapplied function references, extending property wrappers to all patterns is still a viable future direction.

Enabling the application of property wrappers in value-binding patterns would facilitate using the intuitive property-wrapper syntax in native language constructs, as shown below:

```swift
enum Review {
  case revised(Traceable<String>)
  case original(String)
}

switch Review(fromUser: "swiftUser5") {
case .revised(@Traceable let reviewText),
     .original(let reviewText):
  // do something with reviewText
}
```

### Property-wrapper parameters in memberwise initializers

Synthesized memberwise initializers could use property-wrapper parameters for stored properties with attached property wrappers:

```swift
struct TextEditor {
  @Traceable var dataSource: String

  // Synthesized memberwise init
  init(@Traceable dataSource: String) { ... }
}

func copyDocument(in editor: TextEditor) -> TextEditor {
  TextEditor($dataSource: editor.$dataSource)
}
```

### Support `inout` in wrapped function parameters

This proposal doesn't currently support marking property-wrapped function parameters `inout`. We deemed that this functionality would be better tackled by another proposal, due to its implementation complexity. Nonetheless, such a feature would be useful for mutating a `wrappedValue` argument when the wrapper has a `mutating` setter.

### Wrapper types in the standard library

Adding wrapper types to the standard library has been discussed for types [such as `@Atomic`](https://forums.swift.org/t/atomic-property-wrapper-for-standard-library/30468) and [`@Weak`](https://forums.swift.org/t/should-weak-be-a-type/34032), which would facilitate certain APIs. Another interesting standard library wrapper type could be `@UnsafePointer`, which would be quite useful, as access of the `pointee` property is quite common:

```swift
let myPointer: UnsafePointer<UInt8> = ...

myPointer.pointee 
//        ^~~~~~~ 
// This is the accessor pattern property 
// wrappers were devised to tackle.
```

Instead of writing the above, in the future one might be able to write the following:

```swift
let myInt = 0

withUnsafePointer(to: ...) { $value in
  print(value) // 0
  
  $value.withMemoryRebound(to: UInt64.self) {
    ... 
  }
}
```

As a result, unsafe code is not dominated by visually displeasing accesses to `pointee` members; rather, more natural and clear code is enabled.

## Revisions

### Changes from the first reviewed version

* Passing a projected value using the `$` calling syntax is supported via `init(projectedValue:)`.
* The type of the unapplied function reference uses the wrapped-value type by default. Referencing the function using the projected-value type is supported by writing `$` in front of the argument label, or by writing `$_` if there is no argument label.
* Closures with property-wrapper parameters have the same semantics as unapplied function references.
* Additional arguments in the wrapper attribute are supported, and these arguments have the same evaluation semantics as default function arguments.

## Appendix

#### Mutability of composed `wrappedValue` accessors

The algorithm is computing the mutability of the synthesized accessors for a wrapped parameter (or property) with _N_ attached property wrapper attributes. Attribute 1 is the outermost attribute, and attribute _N_ is the innermost. The accessor mutability is the same as the mutability of the _N_ th .wrappedValue access, e.g. _param.wrappedValue<sub>1</sub>.wrappedValue<sub>2</sub>. [...] .wrappedValue<sub>N</sub>

The mutability of the _N_ th access is defined as follows:

* If _N = 1_, the mutability of the access is the same as the mutability of the wrappedValue accessor in the 1st property wrapper.
Otherwise:
  * If the wrappedValue accessor in the _N_ th property wrapper is nonmutating, then the _N_ th access has the same mutability as the _N - 1_ th get access.
  * If the wrappedValue accessor in the _N_ th property wrapper is mutating, then the _N_ th access is mutating if the _N - 1_ th get or set access is mutating.

**Example**: Consider the following `Reference` property wrapper, which is composed with `Asserted` and used on a function parameter:

```swift
@propertyWrapper
struct Reference<Value> {
    
  var wrappedValue: Value {
    get 
    nonmutating set
  }
    
  var projectedValue: Reference<Value> {
    self
  }
  
}

func useReference(@Reference @Asserted(.nonEmpty) reference: String) {
  ...
}
```

In the above example, the function `useReference` is equivalent to:

```swift
func useReference(reference _reference: Reference<Asserted<String>>) {
  var reference: String {
    get { 
      _reference.wrappedValue.wrappedValue
    }
    set { 
      _reference.wrappedValue.wrappedValue = newValue
    }
  }

  var $reference: Reference<Asserted<String>> {
    get {
      _reference.projectedValue
    }
  }
    
  ...
}
```

Since both the getter and setter of `Reference.wrappedValue` are `nonmutating`, a setter can be synthesized for `var reference`, even though `Asserted.wrappedValue` has a `mutating` setter. `Reference` also defines a `projectedValue` property, so a local computed property called `$reference` is synthesized in the function body, but it does _not_ have a setter, because `Reference.projectedValue` only defines a getter.

## Acknowledgements

This proposal was greatly improved as a direct result of feedback from the community. [Doug Gregor](https://forums.swift.org/u/douglas_gregor) and [Dave Abrahams](https://forums.swift.org/u/dabrahams) surfaced more use cases for property-wrapper parameters. [Frederick Kellison-Linn](https://forums.swift.org/u/jumhyn) proposed the idea to change the behavior of unapplied function references based on argument labels, and provided [ample justification](#passing-a-property-wrapper-storage-instance-directly) for why the semantics in the first revision were unintuitive. [Lantua](https://forums.swift.org/u/lantua) pushed for the behavior of closures to be consistent with that of functions, and proposed the idea to use `$` on closure parameters in cases where the wrapper attribute is unnecessary. Many others participated throughout the several pitches and first review. This feature would not be where it is today without the thoughtful contributions from folks across our community.
