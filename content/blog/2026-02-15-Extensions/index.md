+++
title = 'Over-Extended Types. On the overuse of Swift Extensions.'
date = '2026-02-15'
draft = false
+++

Sometimes I see one thing in code, and it bugs me. An excessive love for Swift extensions. We extend everything left, and right.

<!–more–>

Let's start with a synthetic example, then work our way to real-world cases.

```swift
struct Cat {
  func sleep()
  func eat()
  func poo()
  func play()
  func kernelPanic()
}
```

An ordinary cat. A model object in our app. We've covered it with tests, it's SOLID down to the last letter.

We're building the app. Someone is in charge of analytics. The product team says we need to log how many times the cat pooped (we're probably going to sell litter). In the analytics module, an a burst of OOP thought hits:

```swift
extension Cat {
  func propertiesForAnalytics() -> [String: Any] {
    [ "Poops Count": self.poopsCountToday() ]
  }
}
```

Work continues. The team is writing UI. Designers say that cats who rolled in rotten fish (your cat never roamed the back alleys of Odesa?) should be drawn in green:

```swift
fileprivate extension Cat {
  func cellColor() -> UIColor {
    if self.fellVictimOfTheViciousFishOdour() {
      return .green
    }

    return .separatorColor
  }
}
```

Things are moving fast. We're working on improving the first-time user experience. If the user doesn't have a cat yet, we show stubs of popular cats from the market:

```swift
extension Cat {
  static func makeStub(named: String) -> Cat {
     // make a cat and attach a flag with a "pure Swift technology" named associated object
  }

  func isStub() -> Bool {
    // read some stuff attached to the cat with a piece of scotch tape
  }
}
```

The example is purely synthetic, but it captures the idea. You'll say – we don't write code like that! Sure, sure...

Here are the problems I see.

1. Single Responsibility violation. Should a cat know anything about analytics or the color of a table cell? Everyone will say – no! And yet I keep seeing code like this.
2. Interface bloat. Cat used to have five methods. Now it has eight.
3. The interface and implementation are spread across the entire project. I'm new to the project. I read the Cat file, and everything seems clear – eat, sleep, poop – all logical. But nope. Then I suddenly discover that the cat has mysterious hidden abilities of logging itself to analytics. Is there anywhere I can see the full interface of this object?
4. Refactoring. Say we decide to refactor Cats (a lost cause, but anyway). We get together for grooming, look at the Cat, and estimate it at 2 story points. Someone sits down to work, and then all the cat's extension horcruxes we scattered across the project bite them in the rear. Suddenly we also need to rewrite the analytics logic and the stubs! Bet you didn't see that hairball coming, huh?
5. Similar to the above, but slightly different. This smearing breeds islands where people keep piling extensions on top of extensions. Why not – someone already dumped stuff here, why should I be any better? And so next to cellColor() you get avatar(), downloadProfile(), and all of it on Cat (where else would it go?).
6. Tests. What tests? Where the Cat was originally born, there might have been some impulse to write tests, and something got covered. But this is just an extension... I have a task, we have a release. What could go wrong with one little method?!
7. For the extension to access the type's internals, access levels get lowered. `private` becomes `internal`, and sometimes (we're all friends here) it can even escalate to `public`.


# Let's move on to real-world examples

## VirtualFile

This example is delightful because it never made it to production.

```swift
public extension VirtualFile {
  var isTemporaryFile: Bool {
    self.fullPath.hasPrefix(TempFileSystem.rootForCurrentSession())
  }
}
```

Let's discuss whether this method deserves a place in the public interface of VirtualFile. VirtualFile is an interface for files from any file system. So you can literally walk up to a file on Google Drive and ask it – "are you temporary?" And what's it supposed to say? First, it can't even override isTemporaryFile because it's in an extension (you could hack around it with Obj-C, but we wouldn't do that, would we?). Second, in the context of Google Drive VFS, the concept of isTemporaryFile doesn't even exist – so what would we answer? "false"? "we don't know"? "well yes, but actually no"? "I'm a teapot"? But we have a feature, we have a bug – we need to know if it's temporary. So let's start from there – we need, in a specific place, for a specific case, to check a specific file type to see if it lives in a specific folder – that's absolutely not a public extension on the entire protocol! Write a cozy little `private func` where you need it, and it solves your problem.

```swift
var isSampleFile: Bool {}
```

Exactly the same story. A specific case of a specific problem.

Or take a look at this!

```swift
extension Error {
    var userRejectedAccessToCamera: Bool {
        ... some digging in NSError ...
    }
}
```

Somewhere super local, fileprivate, it might still be somewhat acceptable. But people are people. Someone will have a thousand reasons not to want to think, and that fileprivate becomes perfectly visible to the entire project, and bam! Before you know it, Xcode is dumping a list of twenty suggestions on `Error.` that you... at the very least don't need. And that's the good scenario – sometimes they actively get in the way. You wanted to write Statistics.log(event), but autocomplete slipped in Statistics.log(nicheLibraryEventType), and then you're staring at it bug-eyed like – "Xcode/Swift (pick your favorite) – have you lost your mind? What do you mean I can't log my event? What do you mean wrong type?"


## Collection extensions where ...

Another common case – an extension on Array that contains certain types.

```swift
extension Array where Element: CGPoint {
    func perimeter() {
        // some code that was supposed to calculate the perimeter of a polygon for the author's task
    }
}
```

First thought: why is any Array of CGPoints necessarily a polygon? Could it not be a point cloud – a list of restaurants on a map? Or base points for face recognition?

Second, if we have a task involving polygons, let's just say so in the code. For starters, we could at least do this:

```swift
typealias Polygon = Array<CGPoint>

extension Polygon {
    func perimeter() -> CGFloat {
        //
    }
}
```

This is already better... But there are problems:
- everywhere we pass a Polygon, you can shove in a random cloud of points, and Swift won't even flinch
- our polygon will have everything from Array poking out of it
- if we want to change the backing storage or make optimizations, getting rid of Array will be very painful
- or how about this – our polygon has a requirement that the last segment may or may not be explicitly closed (we're anarchists like that). So suddenly this "generic – works for everyone" extension has super-specific logic for our particular polygon

Therefore:

```swift
struct Polygon {
    private var points: [CGPoint]

    func perimeter() -> CGFloat {
        //
    }
}
```

Now we're talking. Clients will know exactly as much as we want them to know. You can't sneak in some random array of points instead of a Polygon. Swapping Array for MegaFastPointsArray – easy-peasy! Adding a custom description – there's a place for it! And most importantly – we think "polygon", we write "polygon", not "well it's like this little array of points, you know".

Sometimes this OOP adventure reaches an even higher level, and instead of extension Array where Element: CGPoint, someone writes extension **Collection** where Element: CGPoint. Don't get me started on this one...

## So what should we do?

My main point isn't that extensions are evil and I'm against them in principle. No. What matters most to me is module boundaries and domain purity. If an extension adds functionality to an object within the same domain, I wholeheartedly support it. Take CGPoint, for example.

```swift
extension CGPoint {
    func translated(_ x, CGFloat, _ y: CGFloat) -> CGPoint { .. }
}
```

What a gorgeous function!

But if (very much pulled out of thin air):

```swift
extension CGPoint {
    func isInsideView(_ view: UIView) -> Bool {}
}
```

Now I'm sad. The UIView domain sits above the CoreGraphics domain. We've suddenly dropped a UFO into the world of CGPoint. isInsideView is a UIKit domain problem, and it should be solved in the UIKit domain:

```swift
extension UIView {
    func containsPoint(_ point: CGPoint) -> Bool {}
}
```

Remember when inheritance was super popular? I feel like something similar is happening now with extensions. I propose we think of an extension not as "I'll just pile something on the side – nobody will notice," but as part of a type's public API. If you moved this extension into the object itself – would it belong there? If not, then you don't need the extension either. Otherwise we turn our code into this:

![House with chaotic extensions - metaphor for extension abuse](extensions-abuse.webp)
