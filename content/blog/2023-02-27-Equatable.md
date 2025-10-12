+++
title = 'A sad story about Swift Equatable'
date = '2023-02-27'
draft = false
+++

This is the story about the cunning Equatable in Swift.

Let's start as advertised. Everything's cool, everything works:

```swift
class Person: Equatable {
    let name: String

    init(name: String) {
        self.name = name
    }

    static func == (lhs: Person, rhs: Person) -> Bool {
        Swift.print(">> Person operator == called")
        return lhs.name == rhs.name
    }

    static func != (lhs: Person, rhs: Person) -> Bool {
        // Ususally I won't implement this operator – compiler will generate one.
        // I just want to add some logs
        // Even if I had to implement !=, I would impelement it via ==.
        // For the sake of logging, let it be really dumb
        //
        Swift.print(">> Person operator != called")
        return lhs.name != rhs.name
    }
}

let john = Person(name: "John")
let jack = Person(name: "Jack")
```

Check. All correct. Let's dive into the real life.
<!--more-->

```swift
john == jack // false
john != jack // true
```

Let's add a tiny bit of OOP

```swift
class Worker: Person {
    let job: String

    init(name: String, job: String) {
        self.job = job
        super.init(name: name)
    }

    static func == (lhs: Worker, rhs: Worker) -> Bool {
        Swift.print(">> Worker operator == called")

        return lhs.name == rhs.name &&
               lhs.job == rhs.job
    }

    // won't implement the !=. Let the compiler do it's job.
}

let johnTheProgramer = Worker(name: "John", job: "iOS Developer")
let johnTheGardener = Worker(name: "John", job: "Gardener")

johnTheProgramer == johnTheGardener // of course it's false
johnTheProgramer != johnTheGardener // drumroll – false too!
```

A little trouble. The base class != was called.

At one hand, it's all correct and logical – there's the protocol, and one must implement both methods for it to work.

On the other hand, standard library contains the default implementation of !=,
and it lulls vigilance of the developer. Even Xcode adds a stub only for ==.
I.e. everything's set up for the screw up.

This can be easily fixed by adding != to the Worker. But this is not the end of the story.


NSObject jumps to the stage.

```swift
// Equatable is inhereted from NSObject here
class Car: NSObject {
    let brand: String

    init(brand: String) {
        self.brand = brand
        super.init()
    }

    static func ==(lhs: Car, rhs: Car) -> Bool {
        return lhs.brand == rhs.brand
    }
}

let toyota = Car(brand: "Toyota")
let toyota2 = Car(brand: "Toyota")

toyota == toyota2 // true
toyota != toyota2 // 💩 true too
```

Actually, nothing new, still the same problem as before.
Just now our base class is the NSObject, and its implementation of
!= is called, and it just compares pointer addresses:

```swift
toyota != toyota  // false – the same pointer
```

And now, the most interesting stuff.

One of the features of == we are advertised is the search in data structures,
for example, in arrays.

Let's give it a try:

```swift
class Phone: Equatable {
    let brand: String

    init(brand: String) {
        self.brand = brand
    }

    static func == (lhs: Phone, rhs: Phone) -> Bool {
        return lhs.brand == rhs.brand
    }
}

// for the iPhone I'll even start the class name with lowercase letter
//
class iPhone: Phone {
    let model: String

    init(model: String) {
        self.model = model
        super.init(brand: "iPhone")
    }

    static func == (lhs: iPhone, rhs: iPhone) -> Bool {
        return  lhs.brand == rhs.brand &&
                lhs.model == rhs.model
    }
}

let iPhoneX = iPhone(model: "X")

let phones = [ iPhone(model: "3G"), iPhone(model: "4"), iPhoneX, iPhone(model: "6") ]

// search seems to work...
phones.contains(iPhoneX) // true

// or does it?...
phones.firstIndex(of: iPhoneX) // 💩 – 0

// we can also take a look at how filter "works":

let oldModels = phones.filter { $0 != iPhoneX }
oldModels.count // 0 (zero)

// the base class == implementation is called in all these tests :)

```


How can we fix this?

I suggest to add a method to compare object in the base class.
Then implement == in the base class by call to the method.

In case of inheritance from NSObject this will be isEqual.

In general, you may call it however you like. isEqual will do too.

This will allow to avoid the following issues:
 - no need to scratch your head about how to call super
 - no need to bother about implementing both == AND != in your class
 - search in collections will work as expected

For example:

```swift
class Animal: NSObject {
    let type: String

    init(type: String) {
        self.type = type
        super.init()
    }

    override func isEqual(_ object: Any?) -> Bool {
        guard let other = object as? Animal else {
            return false
        }

        return self.type == other.type
    }
}

let lizard = Animal(type: "lizard")
let snail = Animal(type: "snail")

assert(lizard == lizard)
assert(false == (lizard != lizard))
assert(false == (lizard == snail))
assert(lizard != snail)

class Cat: Animal {
    let name: String

    init(name: String) {
        self.name = name
        super.init(type: "cat")
    }

    override func isEqual(_ object: Any?) -> Bool {
        if false == super.isEqual(object) {
            return false
        }

        guard let other = object as? Cat else {
            return false
        }

        return self.name == other.name
    }
}

let catTom = Cat(name: "Tom")
assert(catTom == catTom)
assert(false == (catTom != catTom))

let animals = [ Animal(type: "dog"), catTom, Animal(type: "bird") ]

assert(animals.contains(catTom))
assert(1 == animals.firstIndex(of: catTom))
```

We are not the first on the Earth who face this issue:<br/>
 <https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170116/030562.html><br/>
 <https://bugs.swift.org/browse/SR-1729><br/>
 <https://bugs.swift.org/browse/SR-1218><br/>
 <https://stackoverflow.com/questions/28793218/swift-overriding-in-subclass-results-invocation-of-in-superclass-only><br/>
The last one contains a scientific approach to fix this issue.

In general, in most real-life situations, isEqual approach will work well.


Instead of a conclusion.
The same issue is available in C++. Every person who ever tried to make operator == to
be virtual suffered awfully.
30 years passed, we have the new shiny programming language... and the same old issues.
isEqual still rules.
