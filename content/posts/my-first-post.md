---
title: "My First Post"
date: 2022-08-22T17:01:43+07:00
draft: false
tags: ["some", "one"]
---

# Dynamic generic types
## Abstract
Prior the **Swift 5.7** release there is a lack of using protocol with `associatedtype` as a type of arguments of a method rather as a returning type. As well as this release gives a really convenient way to solve that task with [SE-0309](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md "") and [SE-0346](https://github.com/apple/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md ""), this options still not available for any previous iOS rather then iOS/iPadOS 16 and macOS 13 and for Swift language any previous version itself.

> As Apple states itself at appropriate WWDC 2022 video: the only way to get same abilities in previous swift versions — do a lot of boilerplate code e.g. to write as much overloaded methods as it needs to cover all cases.

The way to solve this kind of tasks in prior swift versions is the target of this paper.

## Problem overview
The problem comes from the protocols restrictions on one hand and generics restrictions on another. Let’s dig it a bit.

So let’s say we’re willing to implement method that will be decode any `Decodable` object that i’ll get. This is actually a meaningful task in case that we’re receiving some useful server response wrapped within almost static object, and we want to cover this response with a single type.

Here’s the example of possible received JSON objects:
First object
```JSON
{
    "id": 1,
    "result": {
        "firstKey": 1,
        "secondKey": "second"
    },
    "client": "iOS"
}
```

Second object:
```json
{
    "id": 1,
    "result": {
        "thidrKey": "okok",
        "fourthKey": true,
        "fifthKey": 1231
    },
    "client": "iOS"
}
```

Leaving API design quality out of scope, let’s work with what we’ve got.

So in this case preferred solution would be to implement one generic method and three structures, one for each given result case and one for response itself.

So let’s do it:
```swift
// `result` object for first kind of server response
struct ResultFirstKind: Decodable {
    let firstKey: Int
    let secondKey: String
}

// `result` object for second kind of server response
struct ResultSecondKind: Decodable {
    let thidrKey: String
    let fourthKey: Bool
    let fifthKey: Int
}
```

Everything is going good so far, let’s implement the wrapper type head-on:
```swift
struct Response: Decodable {
    let id: Int
    let result: Decodable
    let client: String
}


func send(uRLRequest: URLRequest, with session: URLSession) async throws -> Response {
    let (data, response) = try await session.data(for: uRLRequest)
    guard 200 == response.statusCode else { fatalError() }
    return try JSONDecoder().decode(APIResponse.self, from: data)
}
```

And here’s we’ve faced huge problem:
> Compiler could not infers type of concrete `result` field in such implementation for exact call. Since it could be **any** decodable type e.g. and there’s not even a hint to which it will be in exact call. So it can’t infers what will be passed to next calls where this `Response` object will be passed

There’s also another issue raises here: `Decodable` protocol is too broad. Like every `String` numeric and maybe some of other types in your code are conforms it, but you’re waiting exact given types to be recieved within server response.

But hopefully the latter one solves easily: `Response` type evolves to follow:
```swift
protocol Result: Decodable { }

extension ResultFirstKind: Result { }

extension ResultSecondKind: Result { }

struct Response: Decodable {
    let id: Int
    let result: Result
    let client: String
}
```

This evolving solves only one of issues — it restricts possible types to be assigned to `result` property to only one that conforms `Result` protocol, but still this code is not compiling and still swift can’t infer type that should be used on method run.

The other way which is seems workable here is to add an`associatedtype` value into a protocol, which is looked like made exactly for this kind of tasks, this solution could look like follow:

```swift
protocol Responsable: Decodable {
     /// ``Result`` are conforms ``Decodable``
    associatedtype Result: Result
    var id: Int { get }
    var result: Result { get }
    var client: String { get }
}

struct Response: Responsable {
    let id: Int
    let result: Result
    let client: String
}
```

So here we’re saying that `Result` type that is the type of `result` property if of `Response` object is one that conforms with `Result` protocol.

This method is actually is the most straightforward and preferable and **yet not compiles prior swift 5.7**. This comes that protocols with `associatedtype` in it could not be used as a constraints in generic methods in any past swift version.

## Problem summary
So let’s provide problem summary:
1. Compiler could not infer non generic type from protocol set as a method parameter type neither its returning type, so we have to tell compiler the concrete type which is covered under generic one in method calling code rather then in method declaration one instead.
2. Protocols with `associatedtype` value could not be used as a constraint for generic method type parameter neither returning in swift version prior 5.7, so we need a way to pass that required restrictions into the call chain somehow to limit possible type to that that we’re awaiting for.

## The solution
Long story short: here’s the solution code.
```swift
// 1
protocol Resultable: Decodable { }

// 2
struct ResultFirstKind: Resultable {
    let firstKey: Int
    let secondKey: String
}

// 2
struct ResultSecondKind: Resultable {
    let thidrKey: String
    let fourthKey: Bool
    let fifthKey: Int
}

// 3
struct APIResponse<Result>: Decodable where Result: Resultable {
    let id: Int
    let result: Result
    let client: String
}

// 4
func send<Result>(uRLRequest: URLRequest, with session: URLSession) async throws -> APIResponse<Result> {
    let (data, response) = try await session.data(for: uRLRequest)
    guard 200 == response.statusCode else { fatalError() }
     // 5
    return try JSONDecoder().decode(APIResponse<Result>.self, from: data)
}

// 6
let result: APIResponse<ResultSecondKind> = try await APIRequest.send(uRLRequest: URLRequest(url: URL("http://google.com")!), with: URLSession.shared)
```
>explicitly declaring `Result` type in function call is **required**.

So let’s examine this bit of code more closely:

1. `Resultable` protocol declaration, this protocol restricts variation of types conforms `Decodable` to types that we’re actually waiting to be in server response.
2. Implementation of both possible `Result` types which could be returned by a server, both of them are conformed both `Resultable` and `Decodable` protocols.
3. `APIResponse<Result>` structure declaration which also declares generic type `Result` with constraint it to only such types that conforms `Resultable` protocol.
4. Generic method `send<Result>(...) ... -> APIResponse<Result>` that returns concrete `APIResponse` type but with yet so far generic property `Result`. So the code so far solves just second problem of given two, and the first is still there and counting.
5. At this line compiler were actually failed previously. Because it can’t decode generic type, it could decode only concrete types, but there’s none yet.
6. Call of `send` method. And this notation `let result: APIResponse<ResultSecondKind> = ... send()` this is the thing. **This longhand notation are required**, because this is the place and the way how and where we tell compiler what concrete type we’re awaiting for exact given call. And by this call compiller could infer what type will be in that exact object `APIResponse<ResultSecondKind>` it’s `ResultSecondKind` obviously.

There’s one more thing yet.

All that way of managing this code is give you a passibility to extend that generic method to literally any amount of types with just one line of code by just conforming any given type to `Resultable` protocol.

And this working even on client side. So if you’re some network library (like us) this solutions gives your users the ability to create their own `Result` types and using them instead of that you’re providing. This is the thing.

```swift
struct ClientImplementationResultSecond: Responsable {
    let thidrKey: String
    let fourthKey: Bool
    let fifthKey: Int
}

// all good
let result: APIResponse<ClientImplementationResultSecond> = try await APIRequest.send(uRLRequest: URLRequest(url: URL("http://google.com")!), with: URLSession.shared)
```

## Limitations
* Again, this method doesn’t work if you’re using shorthand call (it’ll not compiles).
* And this method couldn’t be turned upside down to use it for `encode` some generic type (e.g. create generic request). Or at least we haven’t found it yet, so if you do, please tell us.

## Conclusion
So this is how the lack of generic protocol usage could be bypassed. This method is less required during 5.7 release, because of implementing both [SE-0309](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md "") and [SE-0346](https://github.com/apple/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md "") proposals. But yet there’s (as always) no backward capability provided by Apple, there’s a room for using this solution at least for two more years if you’re indy and for kinda forever if you’re hard tuff enterprise app core team developer.

