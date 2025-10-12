+++
title = "Don't trust your eyes when looking at weak pointers"
date = '2024-09-14'
draft = false
+++

We have some really old piece of code. It's basically a proxy for an NSTimer retained by the timer instead of our precious View Controller that shows the PDF file. The code was written in the manual reference counting times, then semi-automatically translated to ARC and left for good. Turned out that all these years, since the translation to ARC, it had a bug that led to a leak that went under the radar of leak detection tools.

<!--more-->

The code of the timer looks pretty much like this:

```objectivec
@interface MYAutosaveTimer ()
@property (nonatomic, weak) MYViewController *owner;
@end

@implementation MYAutosaveTimer

- (void)setOwner:(MYViewController *)owner {
    if (_owner) {
        // invalidate and release the real timer
    }

    _owner = owner;

    if (_owner) {
        // start the real timer
    }
}

@end
```

The client code looks something like this:

```objectivec
- (instancetype)init {
    ...

    _autosaveTimer = [MYAutosaveTimer new];
    [_autosaveTimer setOwner:self];

    ...
}

- (void)dealloc {
    ...
    [_autosaveTimer setOwner:nil];
}

```

Looks awful in the modern days, but just as I said – an old stuff 🤷‍♂️. I've rewritten it for good now.

The thing that caught my eye, and the reason that I decided to write about this is the way this looks in LLDB. You step into the -setOwner: method. Observe the variables, and the `_owner` is a perfectly fresh and healthy pointer with some address in it. And then you never get into the `if (_owner) { }`, where the resources were supposed to be released.

So, the moral is – don't trust your eyes with the weak pointers to an object that is being deallocated. Pointers are not just plain old addresses these days.
