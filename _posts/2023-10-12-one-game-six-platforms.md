---
layout: post
title: "One Game, By One Man, On Six Platforms: The Good, The Bad and The Ugly"
---

Recently, Valve announced that Counter-Strike 2 would [no longer support macOS](https://www.macrumors.com/2023/10/10/valve-confirms-counter-strike-2-no-macos/). As a one-man indie developer who has released [a game supporting macOS](https://store.steampowered.com/app/1574000/Industry_Idle/), I find myself at first surprised at Valve's decision. But after looking back at my own journey of supporting six platforms, I feel I can understand Valve's perspective.

I feel it is beneficial to write down some of my "lessons learned", hopefully, that will help fellow indie developers choose which platforms to support. A bit of background, my own game [Industry Idle](https://store.steampowered.com/app/1574000/Industry_Idle/) is mostly built on web technologies (WebGL + TypeScript). This means supporting different platforms is comparatively easy - I don't have to deal with platform-specific graphics APIs (DirectX, OpenGL, Vulkan, Metal) and I mostly live inside the browser sandbox. This is pretty much the sunny day scenario when it comes to cross-platform support. Yet, I am constantly surprised and plagued by platform-specific issues.

## Web

### The Good

If I am asked to choose a "first-class" platform for Industry Idle, it would be the web. During development, I am running the game in my web browser. So supporting the web is pretty much "automatic". The platform itself does not need much hassle: I can easily set up an automated build pipeline that deploys to Github/CloudFlare pages, which is mostly free.

### The Bad

As a platform to release a game, the web platform is very saturated. There are a few portal websites left but they are geared towards a very casual audience, who would not play a hard-core factory builder and economy simulation game. The web version makes very little money. If Industry Idle is written without web-first support, I would probably not port the game to the web at all.

### The Ugly

It is very easy to cheat on the web - after all, the nice development and debug tooling that I enjoy also makes cheating very easy. I have to implement a login system (I implemented Steam OpenID login) so that I can ban bad actors - the game has a multiplayer player trade system and a cheater can easily ruin the experience for everyone. Also, it is possible that people will "steal" the game and host it on their website. I am not really bothered by this as long as people playing on those websites do not cheat.

**TL;DR: Not much work to support but not many players to gain**

## Windows (Steam)

### The Good

Steam is the only storefront where the game is released. I've submitted to Epic Game Store as well but never gotten any reply - maybe it is a good thing as integrating another SDK is probably not worth it. In 2023, Windows is still the dominating platform for gamers. About 99% of all desktop players play on Windows. The platform has a relatively stable API - Microsoft almost never breaks backward compatibilities. Setting up an automated build pipeline is relatively easy - I build the Windows version on my Linux server.

### The Bad

The game uses Electron as the runtime. But in order to integrate with Steam SDK (which is in C++), I have used a few solutions: [greenworks](https://github.com/greenheartgames/greenworks) (no longer maintained), Node-API (via [node-addon-api](https://github.com/nodejs/node-addon-api), requires setting up a cross-platform C++ toolchain and write bindings manually), [Koffi](https://koffi.dev/) (an FFI module for Node.JS, provides precompiled binaries but requires writing bindings manually) and finally [steamworks.js](https://github.com/ceifa/steamworks.js) (calling Steamworks via Rust bindings).

### The Ugly

There are relatively few problems compared to the number of players but occasionally I have to deal with really obscure issues like an old Visual C++ Redistributable installed or a "trimmed" version of Windows missing some DLLs. And a lot of Windows players are not tech-savvy, which makes debugging hard. But I cannot really complain about it - looking at the _percentage_ of people who have encountered platform-specific issues, Windows is probably the lowest among desktops.

**TL;DR: Not supporting Windows is not an option but luckily it's relatively easy**

## macOS (Steam)

### The Good

When I wrote the first line of code of Industry Idle, I was using a Mac. I've been a long-time Mac user and have only migrated to Windows after doing more game dev. Mac accounts for less than 1% of all desktop players but oftentimes people are pleasantly surprised that the game works on Mac and I've got several "thank-you" emails about it. Another good thing about the platform is that the x64 build more or less "just works" on Apple Silicon Macs - I don't have an Apple Silicon Mac and the game only has an x64 build but several "thank-you" emails I've got are from Apple Silicon Mac owners - I guess they should really thank Rosetta. Running in emulation mode is not really ideal but the game runs on a potato so the powerful Apple Silicon can easily handle it even with emulation overhead.

### The Bad

Mac is one of the most problematic platforms, even after Electron handles most of the platform-specifics, there are lots of gotchas:

- [Code signing](https://developer.apple.com/documentation/security/hardened_runtime) with a hardened runtime is almost a must.
- However, Steamworks SDK loads `dylib`, which requires several [entitlement exceptions](https://developer.apple.com/documentation/bundleresources/entitlements/com_apple_security_cs_disable-library-validation). It's not well-documented so good luck figuring out which ones. And fingers crossed that Apple will not one day disallow those exceptions.
- After signing the executable, I need to notarize it. The old notarization API takes forever (for a 200MB executable, anywhere between 30 minutes to several hours), fails randomly, and almost never gives meaningful error messages. The new "notarytool" is much improved, but still adds 10mins to the build pipeline. In comparison, the total "Windows + Linux + macOS + upload to Steam" pipeline without code signing and notarization takes about 2-3 minutes.
- Of course, the above has to be done on a Mac (some limited experimental non-Mac support starts to become [available](https://crates.io/crates/apple-codesign)), which is notoriously bad for build automation. I have a bunch of hacks like `security unlock-keychain` in the build script but it still fails occasionally.
- And, keep in mind, this is the _sunny day scenario_: no platform-specific API porting is needed (like from DirectX/Vulkan to Metal).

### The Ugly

The economy of supporting Mac as an indie is bleak. A Mac is pretty much needed and they are not cheap. All the revenue from the platform does not even cover 10% of the cheapest Mac Mini. And then there's the $100 per year developer membership fee for notarization. Luckily it can be used for iOS as well. Also, Apple is more willing to break backward compatibilities. Sometimes there are new macOS/XCode version requirements, which means upgrading the operating system (takes 1-2 hours, can occasionally fail) and XCode (takes 2+ hours, I usually just leave it overnight).

**TL;DR: Not worth the time and money and definitely not worth the pain**

## Linux (Including Steam Deck)

### The Good

Linux is not picky about hardware - it runs on everything and costs nothing. I run Arch Linux inside a virtual machine on my development machine to test the game. I also run Ubuntu on my build machine to produce the executables. And the server runs on a VPS that runs Debian. Linux is very easy to automate.

### The Bad

Linux is very saturated and the operating systems come in different shapes and forms. Initially, I struggled to get the game running at all. In fact, even an empty Electron app (from the official example) does not run. And after hours of Googling, adding `--no-sandbox` argument solves for _some_ users but not for all. This is kind of expected because it is impossible to know what libraries are installed on the OS and which version. I wish Steam would allow me to say "Linux support is provided at a best-effort basis" but that's not an option - I have to tick the checkbox to provide the Linux build, which sometimes sets the wrong expectation.

After Steam Deck was released, I thought supporting it would be automatic - after all, I have a Linux build - but no. It turns out that unless the game is explicitly marked (by Valve reviewers), Steam Deck will use the Windows build + Proton even if a Linux version is available. And the Linux version is running against ["Steam Runtime"](https://github.com/ValveSoftware/steam-runtime) - a relatively fixed set of libraries that developers can target. It is a good solution to the saturated Linux library situation but unfortunately, several libraries are missing to run Electron. To give Valve credit, they are very responsive and helpful in eventually [resolving the issue](https://github.com/ValveSoftware/steam-runtime/issues/579).

*Update: The above is inaccurate. Steam Deck will use Linux build if it runs on Steam Deck. The problem I've encountered is that Steam Deck cannot run the native Linux build due to missing libraries therefore the Windows build is selected. Thanks for a reader's comment for point it out. It's been a while since I dealt with this issues and some of the details have slipped my mind*

### The Ugly

Linux accounts for less than 1% of the total players, so from a business perspective, it is not worth it. And the platform has the _highest_ percentage of players who run into platform-specific issues - the time used for customer support only makes the economy worse. Also, I don't own a Steam Deck, so adding support is not always easy: Valve does not provide a SteamOS image to test. So Steam Deck-specific code (like controller support, starting the game in full-screen, making the UI bigger, etc) is not tested end-to-end. I've used [HoloISO](https://github.com/HoloISO/holoiso) with limited success.

**TL;DR: Even worse economy than Mac but less annoyance perhaps**

## iOS

### The Good

Industry Idle's core engine is actually a mobile engine - before Industry Idle, I was mainly making mobile games. However, Industry Idle is a desktop-first game, which means porting to mobile requires quite some UI adjustments. The game runs inside a custom `WkWebView` on iOS, with some bridging code that exposes native APIs (like Game Center). iOS hardware and software are generally less saturated compared to Android, which means after testing the game on my aging iPhone 6, I have high confidence that the game will work fine on most other iOS devices.

The base game of Industry Idle is free-to-play, with two Expansion Packs available for purchase (one-time purchase, not subscriptions) - the same as Steam. There's only one optional "reward video" ad that allows a player to double the offline income by watching an ad. The game has no microtransactions - I don't like mobile games that are full of ads or are very pushy about microtransactions.

iOS is generally a more "premium" mobile platform compared to Android, which means more players have purchased expansion packs compared to Android. Even though iOS accounts for only about 30% of all mobile players. it brings in about 50% of all mobile income. Also a nice side-effect of Apple's walled garden is that iOS has the lowest _percentage_ of cheaters.

I know there have been lots of complaints about App Review on iOS. I feel the situation has been much improved. Review time is relatively short (1-2 days) and if a reviewer rejects an update for nonsensical grounds, a quick reply usually solves the problem. For example, once an update was rejected for missing a "Restore Purchase" button. I replied by telling the reviewer where to find the button with a screenshot and also _gently_ pointing out the fact that that button had been there since the initial release, which was more than _50 releases ago_. The update was approved the next day. During the initial release, the game was rejected for "copying an existing game". It turned out the AppStore reviewer believed I was "pirating" the Steam version of Industry Idle. I ended up putting a secret web page on the official website, telling the reviewer that I was indeed the developer behind the Steam version. The release was timely approved. The fact that AppStore has human reviewers is a double-edged sword. On the one hand, people make mistakes, which causes unnecessary annoyances. On the other hand, it's easy to talk to an actual human and resolve any issues. Compared to Google Play, which is very machine-driven and is very hard to find a human for solutions.

### The Bad

`WkWebView` is the only allowed web view on iOS and its update is coupled with the iOS update. Apple stopped supporting the iPhone 6 a while ago, which means it is stuck with iOS 12 and a relatively ancient WkWebView. Luckily most players use a much newer iPhone, which helps with this problem.

Similar to developing for macOS, a Mac is pretty much required for developing for iOS and there's the $100 per year developer membership fee. I think the combined income of both iOS and macOS (95% of which comes from iOS) barely covers the cost of the membership fee and the cheapest Mac Mini. And presumably an iPhone is needed for testing, which means the game would be running at a loss. Luckily I had my old Macbook and iPhone so I didn't have to spend that money.

Just like macOS, iOS is not really designed for build automation either. Partially because I have to run it on a Mac, and partially because the automation pipeline is flaky. [Fastlane](https://fastlane.tools/) is a tremendous time saver, however, the build will occasionally fail (2FA required or Apple Server down) and require manual intervention. Because of this, I don't produce iOS or macOS beta builds.

### The Ugly

Mobile is generally a very different platform compared to desktop. Even though it has a very large audience, reaching the target audience usually requires paid user acquisition (i.e. spending money to buy ads). I have zero marketing budget: a lot of players learn about the game on other platforms; a lot of players join via their friends (word of mouth); and there was once a spike in downloads which I could only imagine was from AppStore Editorial or algorithm. Since I was never told anything about this, I could not confirm it.

The lack of annoying ads and pushy microtransactions provide a good experience on mobile. However, this means the mobile income is abysmal: it's a fraction of desktop versions, even though the player amount is much higher.

**TL;DR: 50% of supporting mobile is supporting iOS**

## Android

### The Good

Android version uses the built-in `WebView` and Google has been doing a decent job of keeping it up-to-date. Since it is _not_ tied to the OS update, a bigger percentage of players on Android have a relatively recent version. However, the same cannot be said about the operating system itself and hardware, which I will get to later. Android's automated build pipeline (powered by Fastlane) is much faster and more stable - it runs on Windows, Mac and Linux and the command line has first-class support. Android devices are generally cheaper to acquire - I use Android as my daily phone so extra saving for me! As mentioned before, Google Play uses lots of machine-based checks, which means update review is generally faster (a few hours) if everything goes well.

### The Bad

Android OS and hardware is a mess! There are lots of different hardware, each can run a different OS version, which can have weird customizations from manufacturers. WebView is in general okay but native APIs are very messy - the same API can behave very differently on different devices (e.g. when display cutout API was first introduced, the adoption was bad and each manufacturer has its own quirks). My strategy is simple: I test on my own phone and cross my fingers hoping it will work on other devices.

Also Android has a relatively aggressive API deprecation strategy. Google Play requires a relatively recent `targetSdkVersion` and would reject an app update if one of the third-party SDKs is deprecated. This means I have to constantly fire up Android Studio and update Android and SDK versions (sometimes Kotlin and Gradle versions as well). The project usually will fail to compile, which means several hours of fixing and testing. This does not happen every day but often enough to be annoying. In comparison, I only have to open XCode and update the native Swift code maybe twice since the initial release.

### The Ugly

Android accounts for 70% of all mobile players but the revenue is about the same as iOS. Few people have purchased the expansion packs and most income actually comes from the one and only _optional_ reward video ad. Actually, the ad implementation on Android is probably buggy since it oftentimes does not show up after backgrounding the game until a hard restart - I have got complaints from users about this! However, I never managed to figure out why - partially because I couldn't reproduce it on my phone and partially because I'd like to spend my time working on the game, not debugging some buggy Ad SDK.

**TL;DR: The other 50% of supporting mobile is supporting Android**

## Conclusion

I appreciate that you've read through my long rant about platform support as a one-man indie and hopefully, this will help you in your decision-making - I know I will definitely make better choices for [my next game](https://www.cividle.com/). When talking about game development, the boring details above are rarely mentioned, yet they are of vital importance in actually shipping a game and they can take a lot of effort. In fact, apart from doing platform support work, most of my time is spent on customer support, community management, reviewing and banning cheaters, and server maintenance. So much so that the work on my new game is severely delayed. I've yet to find a good balance but I am trying my best.

[Discuss on HN](https://news.ycombinator.com/item?id=37862606) or [Reddit](https://www.reddit.com/r/gamedev/comments/176gafm/one_game_by_one_man_on_six_platforms_the_good_the/)