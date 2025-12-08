+++
date = '2016-11-21T10:09:12+01:00'
draft = false
title = 'Insights from my first year as an iOS developer'
+++

About a year ago, in December 2015, I made the switch from frontend web development to iOS app development. I was fortunate enough to be able to do so without joining a new employer, and I was also fortunate enough to work in an all-Swift app. In fact, I’ve not written a single line of Objective-C during my year as an iOS developer.

Before making the switch, I’d worked professionally with the web since 2008, and privately a good bit longer, both with backend and frontend development over the years. I generally like to do a bit of both. But now, it was time to try something new. This blogpost sums up some of my insights after one year as an iOS developer. Let’s get cracking.

### Xcode

Coming from web development, I have mostly been using IntelliJ, Sublime Text and Atom for the last couple of years. Xcode, being a full-scale IDE and not just an editor, is probably closest to IntelliJ in this comparison, and let’s just say JetBrains have set a high standard.

There is no easy way to say this, so I’ll just lay it out flat: Xcode falls short in every aspect when comparing it to IntelliJ. It isn’t better at a single thing. My experience with Xcode is that it’s unstable, slow and unpredictable. Something simple enough as switching between branches on the command-line can make Xcode crash or behave in other unexpected ways. Other times, when something doesn’t work as you’d expect, simply restarting Xcode is regarded as a valid fix. Sometimes, another valid fix is to delete everything in your DerivedData folder. As a developer, I don’t trust Xcode. When is it okay to have an IDE this bad, especially when there is no alternative? Never, Apple, the answer is never. It is never okay to have an IDE that is as bad as Xcode.

![Xcode crashing: Unfortunately a common sight for an iOS Developer](xcode-crash.webp)

An aspect of Xcode I’ve really tried to learn to like is the Interface Builder. Apple seems to be quite fond of the Interface Builder, which I find a bit odd. Coming from web development, I see Interface Builder as a WYSIWYG editor of sorts. And in web development, using any kind of WYSIWYG editor hasn’t been part of professional development for at least 10 years, probably even longer than that. How come Apple is pouring so much effort into it? Coincidentally, while Interface Builder could be the single part of Xcode that receives most attention from Apple, my experience is that it might be the most unstable and unpredictable part of Xcode.

Another topic I’d like to bring up when speaking of Xcode is refactoring support in Swift. It’s quite simple: there is none. Be it that Swift is a new language, but it’s been around for a bit over 2 years now, and there is still no refactoring support for Swift at all. I’ll just repeat that: None at all. Renaming and refactoring code should be dead simple. JetBrains really got this right in my opinion. If you’ve refactored something using IntelliJ, you can trust IntelliJ to have done it, and done it right. Not having refactoring support in your IDE after 2+ years is not okay. Not okay.

If I come across as a bit negative towards Xcode, it’s because I am. It’s unstable, doesn’t have the features I want from an IDE, and there is no alternative. I’m stuck with it.

### Continuous Integration, App Distribution and App Delivery

Apple supports continuous integration in iOS projects with Xcode Server and bots. In our project, we’ve set up a couple of Mac Minis with Xcode Server, using bots and Fastlane to build, test and ultimately deliver apps to iTunes Connect, Apples cloud service for app distribution.

Now, you may ask, what is Fastlane? Fastlane is a set of command-line tools to automate tasks around testing, building and delivering iOS apps. Fastlane is a great tool, surrounded by a great community. But it suffers from not being supported by Apple. It suffers from not being a 1st party tool. You could argue that it’s fundamentally a hack, a product of reverse-engineering private APIs. The fact that Apple doesn’t have a proper solution to this is beyond me. Using Xcode Server and a bot (without using 3rd party tools such as Fastlane), it is impossible to automatically test, build and upload app builds to iTunes Connect. You cannot do it. You have to rely on 3rd party tools to make this work. 3rd party tools that may break whenever Apple decides to change something, because they have to use undocumented features and APIs.

Furthermore, once you have built your app and want to ship it, there is no way to release it gradually to your users, i.e. perform a staged roll-out. Once you’ve released your app, you’ve released it to everyone. And there is no rollback feature. If you happen to release a version that’s really bad in some way, your two options are to fix it ASAP and release a new version, or to remove your app from the App Store entirely while you build a new version.

### Closed-Source Proprietary Environment

A few years ago, I used to do IBM Lotus Domino/Notes development. Basically, it boiled down to using a proprietary IDE to write apps for a proprietary and enterprisey ecosystem. The IDE was horrendous, it was prone to crashing and could never be trusted. There was no SCM support, no build automation tools and no dependency management, just to name a few things. And there was nothing I, as a developer, could do about it. There simply were no alternatives to the solutions (or lack there-of) that IBM provided. I was solely in the hands of IBM.

Perhaps not so strangely, this is what comes to mind when I try to sum up my first year as an iOS Developer: developing for iOS is development in a closed-source proprietary environment. In stark contrast with the web in general, iOS is a closed and proprietary ecosystem where you, as a developer, are in the hands of Apple on so many levels. You rely on Apple for the IDE, developer tools, documentation, dependency management, app distribution channels and so many other things. There is no alternative to the App Store. If you want to change how something in the App Store works or add a new feature to it, you have to rely on Apple to do it for you. For better or for worse, this is how it is.

Apple taketh, and Apple giveth, and there’s not a thing you can do about it. Except file a radar and hope it receives attention.