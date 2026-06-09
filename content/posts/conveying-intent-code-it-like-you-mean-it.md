+++
date = '2018-10-14T00:00:00+02:00'
draft = false
title = 'Conveying intent: Code it like you mean it'
+++

Any given chunk of code is almost always read many times more than it is written. While code is often only written once, it is often read tens, hundreds or thousands of times. This is not new knowledge, it’s been known for quite a while (and it’s not the point of this article). This article is about how we all can make an effort to make our code a little more readable and understandable, with a couple of examples in Swift.

<!--more-->

Code readability can, like a lot of things, be considered a bit like a layered cake (because who doesn’t like cake?). Being able to easily read, understand and comprehend what the code does is the first layer, and if done correctly it will provide a foundation for advanced code readability cakes to be baked on top of it. To bake the first layer of our code readability cake, we need a couple of ingredients. First and foremost, our code must be formatted in a consistent and sensible manner. It also needs to be well-structured into modules and files that make sense.

We also need to give our variables, functions and types sensible names that clearly describe their responsibility and what they do. Giving things sensible names is [notoriously difficult](https://martinfowler.com/bliki/TwoHardThings.html) though, so this should definitely not be underestimated and it is okay to spend time naming things properly. Even the most seasoned developers can get this wrong.

Once we start to get those things right, we can get started on the second layer of our cake: conveying intent in our code. Conveying intent is less about describing what the code does and more about what the code is intended to do. It’s often about nuance, and providing a sense of deeper understanding to the reader. What the purpose of the code is, and how it should be used. How many times haven’t we all wondered, perhaps while trying to resolve a bug, what a particular piece of code was intended to do? What did the developer who wrote it _really mean_?

So how do we make our code convey intent? Well, first off, conveying intent can be hard. Having said that, it’s definitely not rocket surgery. We can all do it. It does require a moment of thought though, and sometimes we need feedback from our fellow developers to learn whether the code we just wrote can be made more readable (and that’s okay). If we know the language and tools we’re using well enough, it definitely makes things easier. Let’s look at a couple of Swift examples.

---

### Example 1 — Conveying intent in data modeling

```swift
struct Engine {
    let cylinders: Int?
    let isElectric: Bool
}
```

In this example, we have created a simple model of an engine. An engine can either be electric or not, and if it’s not electric it has cylinders. It is modeled using a struct, containing a boolean and an optional integer. It allows what was described earlier, plus an invalid combination of an electric engine with cylinders. Even if Engine instances are never created with invalid combinations of properties (like an electric engine with cylinders), it is fully possible to create them, and it needs to be considered throughout the codebase.

Here’s how we could improve it:

```swift
enum Engine {
    case electric
    case combustion(cylinders: Int)
}
```

This version is more narrow, allowing precisely what was just described and nothing more. It uses the compiler to make the invalid combination impossible, and because of this its intent is abundantly clear: if it’s an electric engine, it’s not supposed to have cylinders.

---

### Example 2 — Conveying intent when crashing

```swift
func greetLastPerson(people: [Person]) {
    people.last!.greet()
}
```

This function uses the force unwrap operator to get the last value of the people array, which is like of saying “I know this value is never nil, just give me the value”. If it turns out the the force-unwrapped value *is* nil, the program will crash. Crashing, in this case, is a side-effect of using the force-unwrap operator. Is it intended, or not intended? Who knows!

Let’s see how we can improve:

```swift
func greetLastPerson(people: [Person]) {
    guard let lastPerson = people.last else {
        fatalError()
    }

    lastPerson.greet()
}
```

This function makes it more clear what the author’s intent is: if there are no elements in the people array, the program is *supposed* to crash. It is a design choice. The first function doesn’t convey this information in the same way, even though the result is the same.

---

### Example 3 — Conveying intent in an API

```swift
protocol Service {
    func performWork(completionHandler: (Bool) -> Void)
}

service.performWork { status in
    // what does status mean?
}
```

There’s a service (a protocol in Swift defines a contract, this is sometimes referred to as an `Interface` in other languages, e.g. Java and C#) which performs some kind of asynchronous work, and we pass in a completion handler function to be called when the work is done.

In this example, it is unclear what the boolean being passed into the completion handler function actually means. Does it mean success/error? Well, it might, but how would you know?

Let’s see how we can improve:

```swift
enum Status {
    case success
    case error
}

protocol Service {
    func performWork(completionHandler: (Status) -> Void)
}

service.performWork { status in
    // status is either .success or .error
}
```

In this updated example, the boolean has been replaced with an enum. Even though the enum only has two cases, just like a boolean does, the enum gives us an opportunity to name its cases in a way that clearly describe what they mean.

---

### Example 4 — Conveying intent in data modeling (again)

```swift
protocol FontStyleProvidable {
    var color: UIColor? { get }
    var font: UIFont? { get }
}
```

In this example, there are two optional properties: a color and a font. There are four possible permutations here, but in reality, only two ever occurs: when there’s a color, there’s also always a font (and vice versa). This means there are some illegal states here which could be avoided in data modeling.

Let’s see how we can improve:

```swift
protocol FontStyleProvidable {
    var fontStyle: (color: UIColor, font: UIFont)? { get }
}
```

By changing the two optional properties to an optional tuple, the illegal states are now impossible to represent. If the `fontStyle` property exists, there is always a color and a font. No ambiguity, no extra permutations, and much more intentful.

---

### Conclusion

If we put just a little extra effort into writing our code to make it convey our intent, there’s a huge potential up-side. Imagine how many hours of head scratching and beard stroking we can avoid by writing code the way we mean it.
