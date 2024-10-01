---
layout: post
title: Swift for Backend
subtitle: Never tasted it before
# cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/swift_icon.png
# share-img: /assets/img/path.jpg
tags: [Backend, Swift, Programming Language, Server Side, Concurrency, Vapor, SwiftNIO]
author: Yuchen Jiang
---

## Introduction

This article is a simple report to discuss the feasibility of using Swift to set up a backend server. We will list some frameworks and tools that can be used to build a backend server with Swift.

For different components of a backend server, we will also compare the Swift solution with other popular solutions like Golang or Python.

## Table of Contents

- [Introduction](#introduction)
- [Table of Contents](#table-of-contents)
- [Overview of Backend Swift](#overview-of-backend-swift)
- [Interacting with Databases](#interacting-with-databases)
  - [ORM](#orm)
    - [Connection Pooling](#connection-pooling)
    - [Concurrency Control](#concurrency-control)
    - [Atomic Operations](#atomic-operations)
  - [NoSQL](#nosql)
- [Web Frameworks](#web-frameworks)
- [Multi-threading and Concurrency](#multi-threading-and-concurrency)
- [Data structure for concurrency control \& race condition](#data-structure-for-concurrency-control--race-condition)
- [Swift NIO](#swift-nio)
- [Conclusion](#conclusion)


## Overview of Backend Swift

Comparing to other popular backend languages like Golang, Python, or Node.js, Swift is not a very popular choice for backend development. However, Swift has its own advantages in terms of performance and safety.

- Swift is a compiled language, which means it can be faster than interpreted languages
- Swift is a type-safe language, which means it can prevent many runtime errors
- memory management is handled by ARC (Automatic Reference Counting) in Swift, which can prevent memory leaks
- OOP and functional programming are both supported in Swift, and it also supports closures, generics, tuples and protocols

## Interacting with Databases

### ORM

Several ORM libraries are available for Swift, such as:
- [GRDB](https://github.com/groue/GRDB.swift)
- [Vapor's Fluent](https://docs.vapor.codes/4.0/fluent/overview/)
- [Swift-Kuery](https://github.com/Kitura/Swift-Kuery)
- [Perfect-CRUD](https://github.com/PerfectlySoft/Perfect-CRUD?tab=readme-ov-file)

Since in most cases we'll need vapor as the web framework, we'll focus on Fluent here.

Fluent now officially supports SQLite (won't use in backend in most cases), MySQL, PostgreSQL, and even MongoDB. For other databases, check fluent drivers here: [Fluent Drivers](https://github.com/topics/fluent-driver).  

#### Connection Pooling

Fluent has built-in connection pooling, which can be configured in the `app.swift` file. 

```swift
import Fluent
import FluentPostgresDriver

// use postgres as an example
let config = PostgresConfiguration(
    hostname: "localhost",
    port: 5432,
    username: "username",
    password: "password",
    database: "database_name"
)

// set up connection pool
let dbConfig = DatabaseConfiguration(
    configurations: [
        .postgres(configuration: config)
    ],
    pools: [
        "postgres": .init(
            maxConnections: 10, // max connections
            // add other configurations here
        )
    ]
)

// set up the app
app.databases.use(.postgres(configuration: config, maxConnections: 10), as: .postgres)
```

#### Concurrency Control

Fluent is based on SwiftNIO, which is a non-blocking I/O library. And SwiftNIO is designed to be thread-safe, using EventLoop to handle concurrency.
- connection pooling: connection pool is thread-safe, multiple threads can share the same connection pool
- database operations: database operations are executed on the EventLoop. For a certain connection, all the operations are executed in a serial way.

That means the inner mechanism of Fluent is thread-safe, and also transparent to upper layers. What really matters here is that we should avoid assign blocking operations on the Eventloop thread:

```swift
import Fluent
import FluentPostgresDriver
import Vapor

func routes(_ app: Application) throws {
    app.get("process") { req -> EventLoopFuture<String> in

        // what we should avoid
        // let result = performBlockingTask() // this would block the EventLoop thread
        // return req.eventLoop.future("Result: \(result)")
        
        // what we should do
        return req.application.threadPool.runIfActive(eventLoop: req.eventLoop) {
            performBlockingTask()
        }.map { result in
            return "Result: \(result)"
        }

        // or simply use async/await like what you'll do in JS/TS
    }
}

func performBlockingTask() -> String {
    Thread.sleep(forTimeInterval: 2)
    return "Blocking Task Completed"
}

```

#### Atomic Operations

Fluent supports atomic operations, which means we can use transactions to ensure the atomicity of multiple operations.

```swift
app.db.transaction { db in
    // multiple operations here
    return User.query(on: db).filter(\.$name == "Alice").first()
        .flatMap { user in
            guard let user = user else {
                return db.eventLoop.future(error: Abort(.notFound))
            }
            user.name = "Bob"
            return user.save(on: db)
        }
}
```
### NoSQL

For NoSQL databases, Fluent also supports MongoDB. Or you can use the official MongoDB driver for Swift: [MongoDB Swift](https://mongodb.github.io/mongo-swift-driver/).

## Web Frameworks

[Vapor](https://vapor.codes/) is the most popular web framework for Swift. It is based on SwiftNIO, which is a non-blocking I/O library. 

The basic idea for Vapor and SwiftNIO is that all the operations are executed on the EventLoop, which is a lot alike to Node.js. But the difference is that SwiftNIO is designed based on multi-threaded EventLoop, which means it can better utilize the multi-core CPU. Also, Vapor and SwiftNIO are able to untilize the Swift's built-in concurrency control mechanism, like async/await, GCD(Grand Central Dispatch).

## Multi-threading and Concurrency

So, here are the basic methods to handle multi-threading and concurrency in Swift backend development:

- GCD (Grand Central Dispatch): GCD is a low-level API for managing concurrent operations. 

  - GCD provides a way to perform tasks concurrently, either synchronously or asynchronously.
  - However, queues in GCD are not mapped to threads directly, create a queue doesn't mean create a thread. Number of queues won't affect the concurrency level.
  - System will manage the threads by using thread pool, which is transparent to developers.
  - Serial queues vs. Concurrent queues: the only difference is that serial queues execute tasks in order, while concurrent queues execute tasks concurrently.
- Swift Concurrency Model: Swift 5.5 introduces a new concurrency model, which is based on async/await and actors.

  - async/await: the usage is similar to JS/TS. However, in Swift, async/await will block the current coroutine. 
  - Task: Task is a lightweight unit of work. Task can be executed concurrently, and can be cancelled or paused. Task is like the coroutine in other languages, like goroutine in Golang. Also, Task can be nested.
  - However, Task only indicate the schedule of the work, it doesn't mean the work will be executed directly as goroutine does. The actual execution is still managed by the system, which is transparent to developers.
  - Task Groups: Task groups are used to group multiple tasks together. Like the `Promise.all` in JS/TS or `sync.WaitGroup` in Golang.

So the real problem here is that, comparing to traditional backend languages like Golang or Java, Developers don't need to care or even unable to control the threads management. The low-level concurrency control is handled by the system, and developers can't control any part of the process.

## Data structure for concurrency control & race condition

- NSLock: NSLock is a basic lock mechanism in Swift. 
- DispatchSemaphore: DispatchSemaphore is a semaphore mechanism in Swift. It can be used to control the number of concurrent tasks.

  ```swift
  let semaphore = DispatchSemaphore(value: 1)

  func performTask() {
      semaphore.wait()
      // critical section
      semaphore.signal()
  }
  ```

  ```swift
  let semaphore = DispatchSemaphore(value: 0)

  DispatchQueue.global().async {
      print("Executing task 1")
      sleep(2)
      print("Task 1 done")
      semaphore.signal()
  }

  semaphore.wait()
  print("Task 1 has finished, continue with task 2")
  ```
- Actor: actor is a new concept in Swift 5.5. Actor is a reference type that ensures the safety of its mutable state. 

  - Actor is like a class, but it can only be accessed by one task at a time.
  - Actor is a reference type, which means it can be shared across multiple tasks.
  - Actor uses its built-in mechanism to ensure the safety of its safety.

## Swift NIO

Swift NIO provides Eventloop mechanism for vapor, which has some diffrences comparing to the one we used in Node.js.

Basically, Eventloop in Swift NIO is multi-threaded, and each Eventloop is bound to a thread. As for the number of threads generated for eventloops, it's set to the number of CPU cores by default, like `let eventLoopGroup = MultiThreadedEventLoopGroup(numberOfThreads: System.coreCount)`.

Also for these threads generated, they are seperated from those used for GCD, which means the threads used for Eventloop won't be added to the thread pool of GCD.


## Conclusion

After these discussions and comparisons, it becomes clear that Swift, as a backend language, abstracts many low-level operations, such as direct thread management. However, Swift provides high-level concurrency control mechanisms, like async/await, Task, and Actor, which simplify concurrency management for developers.

This makes Swift an excellent choice for developers who are either familiar with the language or prefer not to manage concurrency at a low level. However, for those accustomed to languages like Golang, Java, or C++, Swift may feel more like a refined tool with limitations. It’s not that Swift is bad or difficult to use, but rather that its lack of direct control can be frustrating for developers seeking more granular control—much like the sleek Apple devices of the 2010s. While beautiful and user-friendly, they sometimes fall short when deeper control is desired. In the end, developers are often looking for a tool, not a toy.

