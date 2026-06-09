+++
date = '2018-10-14T00:00:00+02:00'
draft = false
title = 'Conveying intent: Code it like you mean it'
+++

Any given chunk of code is almost always read many times more than it is written. While code is often only written once, it is often read tens, hundreds or thousands of times. This is not new knowledge, it’s been known for quite a while, and it’s not the point of this article. This article is about how we all can make an effort to make our code a little more readable and understandable, with a couple of examples in Swift.

<!--more-->

Code readability can, like a lot of things, be considered a bit like a layered cake, because who doesn’t like cake? Being able to easily read, understand and comprehend what the code does is the first layer, and if done correctly it will provide a foundation for advanced code readability cakes to be baked on top of it. To bake the first layer of our code readability cake, we need a couple of ingredients. First and foremost, our code must be formatted in a consistent and sensible manner. It also needs to be well-structured into modules and files that make sense.

We also need to give our variables, functions and types sensible names that clearly describe their responsibility and what they do. Giving things sensible names is [notoriously difficult](https://martinfowler.com/bliki/TwoHardThings.html), though, so this should definitely not be underestimated and it is okay to spend time naming things properly. Even the most seasoned developers can get this wrong.

Once we start to get those things right, we can get started on the second layer of our cake: conveying intent in our code. Conveying intent is less about describing what the code does and more about what the code is intended to do. It’s often about nuances, and it’s about providing a sense of deeper understanding to the reader. What the purpose of it is and how it should be used.

How many times haven’t we all wondered, perhaps while trying to resolve a bug, what a particular piece of code was intended to do? What did the developer who wrote it mean?

So how do we make our code convey intent? Well, first off, conveying intent can be hard. Having that said, it’s definitely not rocket surgery. We can all do it. It does require a moment of thought though, and sometimes we need feedback from our fellow developers to learn that the code we just wrote can be made more readable, and that’s okay. If we know the language and tools we’re using well enough, it definitely makes things easier. Let’s look at a couple of Swift examples.

---

### Example 1 — Conveying intent in data modeling

```swift
struct Engine {
    let cylinders: Int?
    let isElectric: Bool
}
```

With clear intent:

```swift
enum Engine {
    case electric
    case combustion(cylinders: Int)
}
```

In this example, we have created a simple model of an engine. An engine can either be electric or not, and if it’s not electric it has cylinders. The first example models this using a struct containing a boolean and an optional integer. It allows what was described earlier, plus an invalid combination of an electric engine with cylinders.

Even if `Engine`s from the first example are never created with invalid combinations of properties, like an electric engine with cylinders, it is fully possible to create them, and it needs to be considered throughout the code base. The second example is more narrow, allowing precisely what was just described and nothing more. It uses the compiler to make the invalid combination impossible, and because of this its intent is abundantly clear: if it’s an electric engine, it’s not supposed to have cylinders.

---

### Example 2 — Conveying intent when crashing

```swift
func greetLastPerson(people: [Person]) {
    people.last!.greet()
}
```

With clear intent:

```swift
func greetLastPerson(people: [Person]) {
    guard let lastPerson = people.last else {
        fatalError()
    }

    lastPerson.greet()
}
```

This example is interesting, because the two functions above work exactly the same and produce exactly the same result. If there aren’t any `Person` objects in the array, the program crashes. Otherwise, they call the `greet` function on the last `Person` object in the array.

The first function uses the force unwrap operator to unwrap the optional property `last` on `Array`, which is kind of like saying “oh this can’t be nil, just give me the value”. If it turns out that the property is nil, the program will crash. The crashing in this case is a side-effect of using the force unwrap operator.

The second function, however, makes it clear what the intent is: if there are no elements in the `people` array the program is supposed to crash. It is a design choice. The first function doesn’t convey this information in the same way, even though the result is the same. Is the force unwrap by design, or is it a mistake? Did the author apply an Xcode Fix It which introduced the force unwrap? There’s no way of knowing.

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

With clear intent:

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

This example is a bit longer. There’s a service, a protocol in Swift defines a contract, this is sometimes referred to as an interface in other languages, for example Java and C#, which performs some kind of asynchronous work, and we can define a completion handler function to be called when the work is done. In the first example, it is a bit unclear what the boolean that gets passed to the completion handler function actually means. Does it mean success or error? Well, it might, but how would you know?

In the second example, the boolean is replaced with an enum. Even though the enum only has two cases, just like a boolean does, the enum gives us an opportunity to name its cases in a way that clearly describe what they mean.

---

### Example 4 — Conveying intent in data modeling (again)

```swift
protocol FontStyleProvidable {
    var color: UIColor? { get }
    var font: UIFont? { get }
}
```

With clear intent:

```swift
protocol FontStyleProvidable {
    var fontStyle: (color: UIColor, font: UIFont)? { get }
}
```

This example is similar to the first one, but it uses a tuple to keep related properties together. Both examples are simple ways of reducing the amount of possible permutations in our data modeling. In the first part of this example, there are two optional properties: a color and a font. There are four permutations of them being nil here, but in reality, only two ever occur: when there’s a color, there’s also always a font, and vice versa.

The second part of the example illustrates how we could improve it using a tuple. If the `fontStyle` property exists, there is always a color and a font. No ambiguity, no extra permutations.

---

### Conclusion

If we put just a little extra effort into writing our code to make it convey our intent, there’s a huge potential upside. Imagine how many hours of head scratching and beard stroking we can avoid by writing code the way we mean it.
