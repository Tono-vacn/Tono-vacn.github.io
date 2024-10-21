---
layout: post
title: Swift Review
subtitle: Prepare for midterm
# cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/swift_icon.png
# share-img: /assets/img/path.jpg
tags: [Swift, Programming Language, SwiftUI, DataModel, SwiftData]
author: Yuchen Jiang
readtime: true
---

## Overview of Swift

- single line of declaration:
  ```swift
  var x = 0.0, y = 0.0, z = 0.0
  var red, green, blue: Double
  ```

- multiline comments:
  Unlike multiline comments in C, multiline comments in Swift can be nested inside other multiline comments. 
  ```swift
  /* This is the start of the first multiline comment.
  /* This is the second, nested multiline comment. */
  This is the end of the first multiline comment. */
  ```

- object types:
  - Classes are passed by reference, which allows for things like inheritance
  - Structs and enums are passed by value

- type alias:
  ```swift
  typealias AudioSample = UInt16
  ```

- computed variables:
  - Variables can be stored or computed ( get and set routines)
  - Stored variables can have *setter observers* ( willSet, didSet )
    ```swift
    var now : String {
      get {
          return NSDate().description
        }
    }
    print(now)

    struct lotsize {
        var acres : Float
        
        init(acres: Float) {
            self.acres = acres
        }
        var sqft : Float {
            get {
                return acres * (200*220)
            }
            set {
                self.acres = newValue / (200*220)
            }
        }
    }

    var myLotSize = lotsize(acres: 3)

    print(myLotSize.sqft)

    myLotSize.sqft = 200000
    print(myLotSize.acres)
    myLotSize.acres = 10
    print(myLotSize.acres)
    print(myLotSize.sqft)
    ```

- Identity Operators:
  - `===` and `!==` are used to check if two objects are the same instance
  - `==` and `!=` are used to check if two objects are equal

- Range Operators:
  - `a...b` is a closed range, including a and b
  - `a..<b` is a half-open range, including a but not b

- Sets also have their own unique Set Operations
  - .intersect, .exclusiveOr, .union, .subtract – return a Set
  - .isSubsetOf, .isSupersetOf, .isDisjointWith – return a Bool

- Value Binding:
  ```swift
  let yetAnotherPoint = (1, -1)
  switch yetAnotherPoint {
  case let (x, y) where x == y:
      print("(\(x), \(y)) is on the line x == y")
  case let (x, y) where x == -y:
      print("(\(x), \(y)) is on the line x == -y")
  case let (x, y):
      print("(\(x), \(y)) is just some arbitrary point")
  }
  ```

- Closures:
  - Closures are reference types – you are creating a reference to the closure, not a new copy or value of the closure.
  - Global and Nested functions are actually 2 types of closures.
  - The 3rd type are called Closure Expressions or “anonymous functions”
    - Global Functions – have a name, don’t capture values
    - Nested Functions – have a name, capture values
    - Closure expressions / anonymous functions – no name, capture values 

- Functions:
  ```swift
  func someFunction(argumentLabel parameterName: Int) { print("inside the function we use parameterName: \(parameterName)")
  }
  ```
  - The argument label is used when calling the function; each argument is written in the function call with its argument label before it.
  - The parameter name is used in the implementation of the function.
  - By default, parameters use their parameter name as their argument label.
  - Optionally, you can use "_" to make the first argumentLabel optional

- Variadic Parameters:
  ```swift
  func arithmeticMean(_ numbers: Double...) -> Double {
      var total: Double = 0
      for number in numbers {
          total += number
      }
      return total / Double(numbers.count)
  }
  print(arithmeticMean(1, 2, 3, 4, 5))
  ```

- In-Out Parameters:  
  ```swift
  func swapTwoInts(_ a: inout Int, _ b: inout Int) {
      let tempA = a
      a = b
      b = tempA
  }
  var a = 5
  var b = 6
  swapTwoInts(&a, &b)
  print(a)
  func hitMe( myHand: inout Int) {
      let newCard = Int(arc4random_uniform(10) + 1)
      myHand += newCard
  }

  var myHand = 3
  hitMe(myHand: &myHand)
  hitMe(myHand: &myHand)
  hitMe(myHand: &myHand)
  print(myHand)
  ```

- Structures:
  - Structures have an implicit initializer, but if there are explicit initializers, then they must ensure all stored properties must be initialized.

- Subscripts:
- Subscripts are shortcuts for accessing the member elements of a collection, list, or sequence.
  ```swift
  subscript(index: Int) -> Int {
      get {
          // return an appropriate subscript value here
      }
      set(newValue) {
          // perform a suitable setting action here
      }
  }
  ```


## More on Swift

- Hashable Protocol:
  ```swift
  extension GridPoint: Hashable {
    static func == (lhs: GridPoint, rhs: GridPoint) -> Bool {
        return lhs.x == rhs.x && lhs.y == rhs.y
    }


    func hash(into hasher: inout Hasher) {
        hasher.combine(x)
        hasher.combine(y)
    }
  }
  ```

- Comparable Protocol:
  - only need to implement the < operator, the rest of the comparison operators are automatically implemented.

- Type Properties:
  - You define type properties with the static keyword. For computed type properties for class types, you can use the class keyword instead to allow subclasses to override the superclass’s implementation.

## SwiftData

- @Model
  - @Attribute
  - @Relationship
    ```swift
    @Model
    class Trip {
      @Attribute(.unique) var name: String  // ensure the name of this Trip is unique in the data model
      var destination: String
      var endDate: Date
      var startDate: Date

      @Relationship(.cascade) var bucketList: [BucketListItem]? = []  // delete all bucketList items when the Trip is deleted
      var livingAccommodation: LivingAccommodation?
    }
    ```

- ModelContainer

  ```swift
  let container = try ModelContainer(
  for: [Trip.self, LivingAccommodation.self],
  configurations: ModelConfiguration(url: URL("path"))
  )
  ```

  ```swift
  import SwiftUI

  @main
  struct TripsApp: App {
    var body: some Scene {
        WindowGroup {
        ContentView()
    }
    .modelContainer(
        for: [Trip.self, LivingAccommodation.self]
    )
    }
  }
  ```

- ModelContext
  ```swift
  struct ContentView: View  {
  @Query(sort: \.startDate, order: .reverse) var trips: [Trip]
  @Environment(\.modelContext) var modelContext

  var body: some View {
    NavigationStack() {
        List {
            ForEach(trips) { trip in
            // ...
                }
            }
        }
      }
  }
  ```

## MVC and MVVM

- MVC:
  - Model: data and business logic
  - View: UI
  - Controller: mediator between Model and View 
- MVVM:
  - Model: data and business logic
    - The data that the application is dealing with
    - Holds the information, but not behaviors or services that manipulate the information (like the font used to display it, etc)
  - View: UI
    - In MVVM, the view is active. As opposed to a passive view which has no knowledge of the model and is completely manipulated by a controller/presenter
    - The view in MVVM contains behaviors, events, and data-bindings that ultimately require knowledge of the underlying model and viewmodel.
  - ViewModel: the abstract representation of the View, using data binding to communicate with the View
    - An abstraction of the view exposing public properties and commands.
    - Instead of the controller of the MVC pattern, MVVM has a binder, which automates communication between the view and its bound properties in the view model.
    - Since the View is active in MVVM, the ViewModel does not require a reference to the View, just bindings to the properties needing update, etc.
- SwiftUI
  - SwiftUI is a declarative UI framework
- ViewControllers 
  - Each View/content hierarchy requires its own view controller, UIViewController
  - Used for interfacing with data model and transitions between views

## PersistentStorage

- Sandbox
  - Documents – set up primarily for doc sharing and backup, so good location for files that you create and want to keep. Backed up / Saved with app.
  - Library - contains Caches sub-directory and Preferences sub-directory
    - Caches – information that you want to persist from session to session but don’t need backing up.
    - Preferences – for Settings info, used by NSUserDefaults.  Backed up / Saved with app.
  - tmp – primarily for temp files, staging, etc

- FileManager and URL objects are used to point to folders
  ```swift
  func getDocumentsDirectory() -> URL {
    let paths = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)
    let documentsDirectory = paths[0]
    return documentsDirectory
  }
  let myFileURL = getDocumentsDirectory().appendingPathComponent("myFile.txt")
  ```
- Codable Protocol
  - Encodable, Decodable and Codable
  - various encoders and decoders can be written to convert your object.  `JSONEncoder` and `JSONDecoder` are examples of this.

## Communication

- Trailing closures
- DispatchQueues
  - Main Queue: A system-provided dispatch queue that schedules tasks for serial execution on the app's main thread.
  - Global Queues: A system-provided dispatch queue that schedules tasks for concurrent execution.
  - Custom Queues
    - Serial Queues: A dispatch queue that executes tasks serially in the order they are added.
    - Concurrent Queues: A dispatch queue that executes tasks concurrently.

- @escaping
  - a keyword used to inform callers of a function that takes a closure that the closure might be stored or otherwise outlive the scope of the receiving function. This means that the caller must take precautions against retain cycles and memory leaks. It also tells the Swift compiler that this is intentional.
  ```swift
  public func delay(bySeconds seconds: Double, closure: @escaping () -> Void) {
    let dispatchTime = DispatchTime.now() + seconds
    DispatchQueue.main.asyncAfter(deadline: dispatchTime, execute: closure)
  }
  ```


  - HTTP Sessions
  - URLSession
    - Completion Handlers
    - Delegate Protocol: allows notification as tasks are done asynchronously



  - URLSessionDataTask: data is provided incrementally to the app as it arrives across the network, as Data
  - URLSessionDownloadTask: data stored in a file and handed to the app
  - URLSessionUploadTask: file is provided by app to upload
- REST: Representational State Transfer, stateless

## Other Concepts

- Property Wrappers
  - Property wrappers are a feature that allows you to attach custom behavior to properties. They are a way to factor out common property patterns into reusable code.
  - @Published, @State, @EnvironmentObject, @ObservedObject, @Binding, @Environment, @FetchRequest

  ```swift
  class DukePerson {
    @propertyWrapper
    struct Email<Value: StringProtocol> {
        var emailProp: Value?
        init(wrappedValue emailProp: Value?) {
            self.emailProp = emailProp
        }
        var wrappedValue: Value? {
            get {
                return validate(email: emailProp) ? emailProp : nil
            }
            set {
                emailProp = newValue
            }
        }
        
        private func validate(email: Value?) -> Bool {
            guard let email = email else { return false }
            let emailRegEx = "[A-Z0-9a-z._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,64}"
            let emailPred = NSPredicate(format:"SELF MATCHES %@", emailRegEx)
            return emailPred.evaluate(with: email)
        }
    }
    
    @Email var email: String?
  }
  var person = DukePerson()
  person.email = "rt113@duke.edu"
  print(person.email!)
  person.email = "joe"
  print(person.email)
  ```

## SwiftUI

- Basic
  - An app follows the “App” protocol and contains a “body” property which follows the “Scene” protocol
  - A view follows the “View” protocol and contains a “body” property which follows the “View” protocol

- @State
  - use the @State property wrapper to designate properties of the view you wish to alter

- @Binding 
  - use the @Binding property wrapper to pass a value from a parent view to a child view
    ```swift
    struct HornButton: View  {
      @Binding var hornPlayedCount: Int

      var body: some View {
          Button("\(hornPlayedCount) times") {
          hornPlayedCount += 1
          }
      }
    }

    struct CarView: View  {
      @State private var hornCount: Int = 0

      var body: some View {
          Text("Your car has honked: ")
          HornButton(hornPlayedCount: $hornCount)
      }
    }
    ```

- Objects
  - ObservableObject: a protocol that allows you to publish changes to your object
  - @ObservedObject: a property wrapper that allows you to observe changes to an object
  - @EnvironmentObject: a property wrapper that allows you to pass an object to a view without having to pass it through the view hierarchy
  - @StateObject: a property wrapper that creates an object and retains it for the lifetime of the view

- Environment
  - @Environment: a property wrapper that allows you to access values from the environment
    ```swift
    struct ContentView: View {
      @Environment(\.colorScheme) var colorScheme
      var body: some View {
          Text("The current color scheme is \(colorScheme)")
      }
    }
    ```
## Graphics and Animation