---
title: Dynamic generic types
date: 2022-10-03T17:01:43+07:00
draft: false
tags: [swift, generics, decodable, API]
---
## Abstract
Prior the **Swift 5.7** release there is a lack of using protocol with `associatedtype` either as a type of passed argument or as a returning type. Prior this release both options to solve such tasks as with [SE-0309](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md "") and [SE-0346](https://github.com/apple/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md "") are not available.

But, you know, this tasks are still exists there. So Apple names its way to solve them in the past:

> do a lot of boilerplate code e.g. to write as much overloaded methods as it needs to cover all cases.

Let’s review another one that solves not all, but some tasks from above.

## Problem overview
The problem comes from the protocols restrictions on one hand and generics restrictions on another. Let’s dig it. 

Let’s say we’re willing to implement method that will be decode any `Decodable` object that i’ll get. This is a meaningful task in case that we’re receiving useful server response wrapped within almost static object, and we want to cover this response with a single type.

Here’s the example of possible received JSON objects:
First object
```json
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

In this case preferred solution would be to implement one generic method and three structures, one for each given result case and one for response itself.

Let’s do it:
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

Everything is going good, let’s implement the wrapper type head-on:
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
> Compiler could not infers type of concrete `result` field in implementation for exact call. Since it could be **any** decodable type e.g. and there’s not even a hint to which it will be in exact call.

## Problem summary
Let’s provide problem summary:
1. Compiler could not infer non generic type from protocol set as a method parameter type neither its returning type, we have to tell compiler the concrete type which is covered under generic one in method calling code then in method declaration one instead.
2. Protocols with `associatedtype` value could not be used as a constraint for generic method type parameter neither returning in swift version prior 5.7, we need a way to pass that required restrictions into the call chain to limit possible type to that that we’re awaiting for.

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
    return try JSONDecoder().decode(APIResponse<Result>.self, from: data)
}

// 5
let result: APIResponse<ResultSecondKind> = try await APIRequest.send(uRLRequest: URLRequest(url: URL("http://google.com")!), with: URLSession.shared)
```
> explicitly declaring `Result` type in function call is **required**.

Let’s examine this bit of code more closely:

1. `Resultable` protocol declaration, this protocol restricts variation of types conforms `Decodable` to types that we’re waiting to be in server response.
2. Implementation of both possible `Result` types which could be returned by a server, both of them are conformed to both `Resultable` and `Decodable` protocols.
3. `APIResponse<Result>` structure declaration which declares generic type `Result` with constraint it to types that conforms `Resultable` protocol.
4. Generic method `send<Result>(...) ... -> APIResponse<Result>` that returns concrete `APIResponse` type but with yet generic property `Result`.
5. Call of `send` method. And this notation `let result: APIResponse<ResultSecondKind> = ... send()` this is the thing. **This longhand notation are required**, because this is the place and the way how and where we tell compiler what concrete type we’re awaiting for exact given call. And by this call compiler could infer what type will be.

There’s one more thing yet.

This solution gives you a passibility to extend that generic method with one line of code by conforming a given type to `Resultable` protocol. 

This works on client side too. If you’re network library (like [we are](https://github.com/skywinder/web3swift#send-erc-20-token)) this is a thing, because your users can pass there their own `Result` types instead of provided by you, which gives a broad flexibility without increasing complexity.

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
* And this method couldn’t be turned upside down to use it for `encode` generic type (e.g. create generic request). Or at least we haven’t found it yet, so if you do, please tell us.

