+++
date = '2024-01-01T09:58:21+01:00'
draft = false
title = 'A technique for managing modal views in iOS'
+++

![Photo by Brett Jordan on Unsplash](https://miro.medium.com/v2/resize:fit:1400/format:webp/0*15sfb55d4VyAXcic)

How often have you been tasked with adding a modal takeover view to an iOS app? You know, one of those views that are supposed to appear when the user launches the app, “on top of everything else”. It’s a common ask, and it can be useful to present something to the user right away. Some examples from my experience include:

1. ‘Time to update!’
2. ‘What’s new in this version’
3. ‘Is this contact information still correct?’
4. Passcode verification when opening the app
5. Display some view because the user opened the app using a deep link

Implementing the actual views for these use cases is one thing, but making sure that they appear at the correct time is another. If we just have one of these modal views, it’s simple: just present it after the app has been launched. But what happens when we want to show multiple modal views upon app launch? Out of the examples above, what happens if the user has passcode verification enabled in the app, opens the app through a deep link, and has just upgraded the app?

In this case the conditions for #2, #4, and #5 (possibly #3 as well) are all met, and in all of those cases a view should be modally presented on top of everything else. In this situation, it’s likely that multiple modal views are presented on top of each other. That might be totally fine. But how do we know what order they are presented in?

Sometimes ordering can be easily controlled, perhaps just by rearranging lines of code in the app. Other times, it will be a bit harder than that. Maybe an HTTP request needs to be sent to know whether a modal should be shown, and only when the response is received do we know if we want to present the modal or not. Additionally, some modal might be more important than others and should always be shown first.

For example, in the example above, we probably want to ask for the user’s passcode before their private contact information is displayed.

How can we bring order to this modal chaos?

### The technique: Modal operations

A technique I learned at a previous employer lets us have fine-grained control over the ordering and sequencing of these modally presented views. It’s quite simple, there’s no framework or library, and there’s really not much to it: it uses an OperationQueue for ordering and sequencing, and each modal view is implemented as an asynchronous Operation which gets added to the queue. Let’s look at some code:

```swift
final class AppUpgradeNudgeModalOperation: ModalOperation {
    override func run(completion: @escaping () -> Void) {
        fetchAppUpgradeNudge { nudge in
            guard let nudge else {
                completion()
                return
            }

            presentAppUpgradeNudgeView(
                nudge: nudge,
                completion: completion
            )
        }
    }
}
```

This is an operation that handles presentation of an upgrade nudge view (‘Time to update!’). The only method that needs to be implemented is the run method. This is where to put the logic for deciding if something needs to be presented or not, and also handle the presentation that might occur. In this case, AppUpgradeNudgeModalOperation tries to fetch a nudge, and if there is one, it gets presented. If not, it calls completion and returns.

⚠️ **Calling `completion` is important**, because this tells the operation queue that this modal operation is finished. If there is a code path that omits the call to completion, this code path would effectively block the queue. This means that it is also important to call completion after the presented view has been dismissed.

If you look closely, you’ll notice that the class above inherits from ModalOperation. This is a base class which allows for the simple API of just a run method with a completion callback. Let’s look at that:

```swift
class ModalOperation: Operation {
    private var _executing = false
    private var _finished = false

    override var isExecuting: Bool { _executing }
    override var isFinished: Bool { _finished }
    override var isAsynchronous: Bool { true }

    override func start() {
        self.willChangeValue(for: \.isExecuting)
        _executing = true
        self.didChangeValue(for: \.isExecuting)

        run(completion: {
            self.willChangeValue(for: \.isExecuting)
            self._executing = false
            self.didChangeValue(for: \.isExecuting)

            self.willChangeValue(for: \.isFinished)
            self._finished = true
            self.didChangeValue(for: \.isFinished)
        })
    }

    func run(completion: @escaping () -> Void) {
        assertionFailure("implement me")
    }
}
```

`ModalOperation` isn’t particularly interesting, it mostly deals with details of Operation, making sure it communicates its asynchronous state to the OperationQueue which it gets added to.

Okay, so now we have `ModalOperation`, of which multiple subclasses can be created. How do we add them to a queue and control ordering? Let’s look at that:

```
lazy var modalOperationQueue: OperationQueue = {
    let operationQueue = OperationQueue()
    operationQueue.maxConcurrentOperationCount = 1
    return operationQueue
}
```

Somewhere in the app we will have to create the operation queue itself. One place to do that could be the `AppDelegate`, but you might want to keep it somewhere else depending on your app and your needs. Anyhow, this is the operation queue to which `ModalOperation `subclasses will be added. Let’s look at that:

```
modalOperationQueue.addOperation(AppUpgradeNudgeModalOperation())
modalOperationQueue.addOperation(WhatsNewModalOperation())
modalOperationQueue.addOperation(UpdateContactInformationModalOperation())
modalOperationQueue.addOperation(PasscodeInputModalOperation())
modalOperationQueue.addOperation(DeeplinkNavigationModalOperation(url: url))swsw
```

As you can see, adding modal operations is a breeze. Ordering is controlled by the order of which modal operations are added to the queue. So in this case, if we want to change the order of modal presentations, it’s a simple matter of rearranging these lines of code.

### Where to go from here?

We were using this technique for years at a previous employer, and we learned that a common source of bugs is that completion isn’t called on all code paths in subclasses of `ModalOperation`. This can lead to some nasty bugs, where some modals are never shown, because their code simply never runs (because the modal operation queue is blocked).

I’ve been using this technique in UIKit-centric apps, and it might not work so well with SwiftUI. I’m not sure.

I’m pretty sure that there are more modern tools than OperationQueue and completion blocks to implement this technique, it could probably just as well be implemented using Tasks and async/await.

### Final remarks

I didn’t come up with this technique, so I can’t take any credit for it. I’m not sure exactly who came up with it, it was already present in the code base when I joined.

Anyhow, if you’ve made it this far into this article, thanks for reading and I hope it gave you something.
