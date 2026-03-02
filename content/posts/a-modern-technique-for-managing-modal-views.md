+++
date = '2025-12-09T09:17:55+01:00'
draft = false
title = 'A modern technique for managing serial modal views'

+++

[In January 2024]({{< relref "/posts/a-technique-for-managing-modal-views-in-ios.md" >}}), I wrote about a technique for managing serial modal views on iOS by subclassing `Operation` and using a serial `OperationQueue`. As it turns out, that technique had its fair share of problems.

First, `OperationQueue` and `Operation` are sharp tools: using them is **hard**. It's easy to cut yourself. Even though I really tried, I never quite got rid of all the problems in my `OperationQueue` based approach, so it kept causing odd problems and hard-to-debug data races. I'm sure it's possible to use this tools properly, I just couldn't get to that point.

Second, `OperationQueue` and `Operation` don't play nice with Swift Concurrency. For example, there is no compile-time isolation checking, so it's easy to accidentally update a property from the wrong context. `Operation` is declared as `@unchecked Sendable`, which means any subclasses also are `@unchecked Sendable`, so already at that point you've opted out of compile-time concurrency checks.

Since introducing this technique in my current project, I've had a long tail of rarely occurring crashes. Crashes I simply couldn't fix. And eventually, I had enough. **There had to be a better way.** And there is ✨.

## The better way™ 

Instead of using `Operation` to model our modal views, we're going to use `Task`. And instead of using an `OperationQueue` for ordering and serial execution, we'll simply hold a list of async functions that are waiting to be run. Sounds easy, right? That's because it is. Let's dive in!

### `LaunchStep`

Each modal operation will conform to the protocol `LaunchStep`:

```swift
@MainActor
protocol LaunchStep {
    var name: String { get }
    func run() async
}

extension LaunchStep {
    var name: String { String(describing: Self.self) }
}
```

It's a simple protocol, containing a name and an async `run` function. Conveniently, the `name` property can be automatically provided by defaulting to the type name of the implementation, so all we have to do is provide a `run` function!

Here's how a launch step fetching an app upgrade nudge might look like:

```swift
final class AppUpgradeNudgeLaunchStep: LaunchStep {
    private let appUpgradeNudgeService: AppUpgradeNudgeService
    private let presentNudge: @MainActor (UIViewController) -> Void

    init(
        appUpgradeNudgeService: AppUpgradeNudgeService,
        presentNudge: @MainActor @escaping (UIViewController) -> Void
    ) {
        self.appUpgradeNudgeService = appUpgradeNudgeService
        self.presentNudge = presentNudge
    }

    func run() async {
        guard let nudge = try? await appUpgradeNudgeService.fetchAppUpgradeNudge() else {
            return
        }

        let nudgeViewController = NudgeViewController(
            appUpgradeNudgeService: appUpgradeNudgeService,
            completion: nil
        )

        guard !Task.isCancelled else {
            return
        }

        presentNudge(nudgeViewController)
    }
}
```

It's quite simple. It uses DI to inject dependencies for fetching and presenting the nudge, and performs all of its fetching and presenting logic in the async `run` method (a requirement of the `LaunchStep` protocol).

Launch steps can contain much more complexity though, here's an example of a launch step presenting a passcode challenge to the user:

```swift
final class PasscodeLaunchStep: LaunchStep {
    private let userPreferences: UserPreferences
    private let present: @MainActor (UIViewController) -> Void

    init(
        userPreferences: UserPreferences,
        present: @MainActor @escaping (UIViewController) -> Void
    ) {
        self.userPreferences = userPreferences
        self.present = present
    }

    func run() async {
        guard userPreferences.value(
            forKey: .isPasscodeEnabled
        ) == true else { return }

        var presentedViewController: PasscodeViewController?
        var delegate: PasscodeViewControllerDelegate?
        let userPreferences = self.userPreferences
        let present = self.present

        await withTaskCancellationHandler {
            await withCheckedContinuation { continuation in
                let viewController = PasscodeViewController.make(
                    userPreferences: userPreferences,
                    context: .unlock
                )

                presentedViewController = viewController
                delegate = Delegate(
                    onUnlocked: {
                        continuation.resume()
                    }
                )
                viewController.delegate = delegate

                guard !Task.isCancelled else {
                    continuation.resume()
                    return
                }

                present(viewController)
            }
        }
        onCancel: {
            Task { @MainActor in
                presentedViewController?.dismiss(animated: false)
            }
        }
    }
}

fileprivate final class Delegate: PasscodeViewControllerDelegate {
    var onUnlocked: () -> Void

    init(onUnlocked: @escaping () -> Void) {
        self.onUnlocked = onUnlocked
    }

    func passcodeUnlocked(
        _ passcodeViewController: PasscodeViewController
    ) {
        onUnlocked()
    }
}
```

This example is a bit more complex, with lots of things happening. It deals with task cancellation, it wraps a delegate into a continuation, and more.

Okay, so now that we've had a look at two launch steps, how do we orchestrate them? It's time to introduce `LaunchSequencer`.

### `LaunchSequencer`

```swift
@MainActor
final class LaunchSequencer {
    private var currentTask: Task<Void, Never>? = nil
    private var pendingSteps: [any LaunchStep] = []

    func enqueue(_ step: LaunchStep) {
        pendingSteps.append(step)
        processQueueIfNeeded()
    }

    func cancelAll() {
        currentTask?.cancel()
        currentTask = nil
        pendingSteps.removeAll()
    }

    private func processQueueIfNeeded() {
        guard currentTask == nil else { return }

        guard !pendingSteps.isEmpty else { return }
        let step = pendingSteps.removeFirst()

        currentTask = Task {
            await step.run()
            stepFinished()
        }
    }

    private func stepFinished() {
        currentTask = nil
        processQueueIfNeeded()
    }
}
```

`LaunchSequencer` is responsible for managing the ordering, serial execution, and cancellation of launch steps. It holds state for the currently executing task (derived from a launch step's `run` method), and a list of pending launch steps.

For adding a new launch step, it provides the `enqueue` method, which when called adds the passed-in launch step to the list of pending steps, and starts processing the queue if necessary.

Upon completion of each launch step, it will clear the current task, and start processing the queue again if needed.

The `cancelAll` method will cancel and nil out the ongoing task (if any), and clear the list of pending launch steps.

### Usage

So, how should we use this? Well, here's how it's used in the app I currently work on:

```swift
private func addLaunchSteps() {
    launchSequencer.enqueue(
        AppUpgradeNudgeLaunchStep(
            appUpgradeNudgeService: ...,
            presentNudge: { ... }
        )
    )

    launchSequencer.enqueue(PasscodeLaunchStep(userPreferences: ...))
    launchSequencer.enqueue(RemoveLaunchScreenLaunchStep())

    if let pendingDeeplink {
        launchSequencer.enqueue(DeepLinkLaunchStep(deeplink: pendingDeeplink))
    }

    if appWasUpgraded {
        launchSequencer.enqueue(WhatsNewLaunchStep(whatsNewService: ...))
    }
}
```

This function is called every time the app is launched or brought into the foreground. It makes sure that these operations run every time, in the same order, when needed.

### Wrapping up

This is a more modern way of dealing with serial modal views in Swift, leveraging compile-time concurrency checks, and is overall simply less code and less error-prone than subclassing `Operation` and managing its different states.

The solution described in this blog post is already live in my app, and is already proving itself by in several ways:

* Isolation enforced at compile time

* No manual state transitions

* Cancellation is explicit and structured

* No reliance on KVO-backed Operation state

If you happen to spot something that can be improved (either in the code I posted, or in this post), please give me a shout out on Mastodon.

#### A note on SwiftUI

I use this an app that mainly uses UIKit at this level, so I can't speak for how well this pattern works for SwiftUI apps.
