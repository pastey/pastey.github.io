+++
title = 'Сумна історія про Swift Equatable'
date = '2023-02-27'
draft = false
+++

Це історія про підступний Equatable в Swift.

Почнемо як в рекламі. Все круто, все працює:

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
        // Зазвичай я не реалізую цей оператор – компілятор згенерує його сам.
        // Я просто хочу додати трохи логів
        // Навіть якби мені довелось реалізувати !=, я б реалізував його через ==.
        // Заради логування нехай буде дійсно тупий
        //
        Swift.print(">> Person operator != called")
        return lhs.name != rhs.name
    }
}

let john = Person(name: "John")
let jack = Person(name: "Jack")
```

Перевіряємо. Все правильно. Перейдемо до реального життя.
<!--more-->

```swift
john == jack // false
john != jack // true
```

Додамо трошки ООП

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

    // не буду реалізовувати !=. Нехай компілятор зробить свою роботу.
}

let johnTheProgramer = Worker(name: "John", job: "iOS Developer")
let johnTheGardener = Worker(name: "John", job: "Gardener")

johnTheProgramer == johnTheGardener // звісно false
johnTheProgramer != johnTheGardener // барабанна дріб – false теж!
```

Невеликі проблеми. Викликався != базового класу.

З одного боку, все правильно і логічно – є протокол, і треба реалізувати обидва методи, щоб воно працювало.

З іншого боку, стандартна бібліотека містить дефолтну реалізацію !=,
і це присипляє пильність розробника. Навіть Xcode додає заглушку тільки для ==.
Тобто все спонукає до факапу.

Це легко виправити, додавши != до Worker. Але це не кінець історії.


NSObject вибігає на сцену.

```swift
// Equatable успадкований від NSObject тут
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
toyota != toyota2 // 💩 true теж
```

Насправді, нічого нового, все та ж сама проблема що і раніше.
Просто тепер наш базовий клас це NSObject, і викликається його реалізація
!=, а вона просто порівнює адреси вказівників:

```swift
toyota != toyota  // false – той самий вказівник
```

А тепер найцікавіше.

Одна з фіч == про яку нам розповідають це пошук у структурах даних,
наприклад, в масивах.

Спробуємо:

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

// для iPhone я навіть почну назву класу з маленької літери
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

// пошук ніби працює...
phones.contains(iPhoneX) // true

// або ні?...
phones.firstIndex(of: iPhoneX) // 💩 – 0

// можемо також подивитись як filter "працює":

let oldModels = phones.filter { $0 != iPhoneX }
oldModels.count // 0 (нуль)

// реалізація == базового класу викликається у всіх цих тестах :)

```


Як це виправити?

Я пропоную додати метод для порівняння об'єктів у базовому класі.
Потім реалізувати == в базовому класі через виклик цього методу.

У випадку успадкування від NSObject це буде isEqual.

Загалом, можна назвати як завгодно. isEqual підійде.

Це дозволить уникнути наступних проблем:
 - не треба ламати голову як викликати super
 - не треба морочитись з реалізацією обох == ТА != у вашому класі
 - пошук у колекціях працюватиме як очікується

Наприклад:

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

Ми не перші на Землі хто стикається з цією проблемою:<br/>
 <https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170116/030562.html><br/>
 <https://bugs.swift.org/browse/SR-1729><br/>
 <https://bugs.swift.org/browse/SR-1218><br/>
 <https://stackoverflow.com/questions/28793218/swift-overriding-in-subclass-results-invocation-of-in-superclass-only><br/>
Останнє містить наукові підхід до вирішення цієї проблеми.

Загалом, у більшості реальних ситуацій підхід з isEqual працюватиме добре.


Замість висновку.
Та сама проблема існує в C++. Кожен, хто коли-небудь намагався зробити operator == віртуальним,
страждав жахливо.
Минуло 30 років, у нас новенька блискуча мова програмування... і ті самі старі проблеми.
isEqual все ще рулить.
