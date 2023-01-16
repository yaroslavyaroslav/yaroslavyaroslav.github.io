---
title: Dynamic generic types
date: 2022-10-03T17:01:43+07:00
draft: false
tags: [swift, generics, decodable, API]
---
## Abstract

Prior to the Swift 5.7 release, there was a lack of support for using protocols with associatedtype as a type of passed argument or as a returning type. Prior to this release, the options to solve such tasks as with [SE-0309](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md "") and [SE-0346](https://github.com/apple/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md "") were not available.

However, these tasks still exist. In the past, Apple's solution was to do a lot of boilerplate code, such as writing many overloaded methods to cover all cases.

Let's review another solution that solves some, but not all of these tasks.

## Problem overview

The problem comes from the restrictions of protocols and generics. To illustrate this, let's consider the task of implementing a method that can decode any Decodable object. This is a useful task when receiving a server response wrapped in object that couldn't be mapped to a JSON key, and we want to cover this response with a single type.

As an example, we have two possible received JSON objects:
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

The preferred solution would be to implement one generic method and three structures, one for each given result case, and one for the response itself.

We can define structs for each type of the result object:

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

and a wrapper struct for the response:

```swift
struct Response: Decodable {
    let id: Int
    let result: Decodable
    let client: String
}
```
However, when we try to implement a method that uses this struct, such as:
```swift
func send(uRLRequest: URLRequest, with session: URLSession) async throws -> Response {
    let (data, response) = try await session.data(for: uRLRequest)
    guard 200 == response.statusCode else { fatalError() }
    return try JSONDecoder().decode(APIResponse.self, from: data)
}
```
we face a problem: the compiler cannot infer the type of the concrete result field in the implementation for an exact call with ``. Since it could be any decodable type, there is not even a hint as to which it will be in the exact call.

## Problem summary
1. In Swift, the compiler cannot infer a non-generic type from a protocol set as a method parameter type or its returning type. We have to explicitly specify the concrete type that is covered under the generic one in the method calling code and in the method declaration.
2. Prior to Swift version 5.7, protocols with associatedtype values could not be used as a constraint for a generic method type parameter or return type. Therefore, we need a way to pass the required restrictions into the call chain to limit the possible types to those that we are expecting.

## The solution
The solution is to use a combination of generics and protocols to provide the necessary type information to the compiler.

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

1. The `Resultable` protocol is declared, this protocol restricts the variation of types that conform to `Decodable` to types that we are expecting to be in the server response.
2. The possible `Result` types that could be returned by the server are implemented. Both of them conform to both the `Resultable` and `Decodable` protocols.
3. The `APIResponse<Result>` structure is declared which declares a generic type `Result` with a constraint to types that conform to the `Resultable` protocol.
4. A generic method `send<Result>(...) ... -> APIResponse<Result>` is implemented that returns a concrete `APIResponse` type but with a yet generic property `Result`.
5. In the call of the send method, explicitly declaring the `Result` type in the function call is required. This notation let result: `APIResponse<ResultSecondKind> = ... send()` tells the compiler what concrete type we are expecting for a given call. And by this call, the compiler can infer the type that will be used.

In this way, we can use the `APIResponse` struct to wrap any type of `Decodable` object and the send method to handle it correctly.

This solution also gives you the possibility to extend the generic method with one line of code by conforming a given type to the Resultable protocol.

This works on the client side as well. If you're using a network library, this is a great feature because your users can pass in their own Result types instead of the ones provided by you, which gives a broad flexibility without increasing complexity.

```swift
struct ClientImplementationResultSecond: Responsable {
    let thidrKey: String
    let fourthKey: Bool
    let fifthKey: Int
}

// all good
let result: APIResponse<ClientImplementationResultSecond> = try await APIRequest.send(uRLRequest: URLRequest(url: URL("http://google.com")!), with: URLSession.shared)
```

Limitations

- This method does not work if you're using shorthand call (it will not compile).
- This method couldn't be turned upside down to use it for encode generic type (e.g. create generic request). At least we haven't found it yet, so if you do, please let us know.
