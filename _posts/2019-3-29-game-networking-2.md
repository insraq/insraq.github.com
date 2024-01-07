---
layout: post
title: "Game Networking Demystified, Part II: Deterministic"
---

Part I: [State vs. Input](https://ruoyusun.com/2019/03/28/game-networking-1.html)  
Part II: [**Deterministic**](https://ruoyusun.com/2019/03/29/game-networking-2.html)  
Part III: [Lockstep](https://ruoyusun.com/2019/04/06/game-networking-3.html)  
Part IV: [Server and Network Topology](https://ruoyusun.com/2019/04/07/game-networking-4.html)  
Part V: [Interpolation and Rollback](https://ruoyusun.com/2019/09/21/game-networking-5.html)  
Part VI: [Game Genres and FAQ](https://ruoyusun.com/2019/09/30/game-networking-6.html)  

Deterministic often appears together with *lockstep* but really they are two separate concept. In the previous part of the series, I state that if you want to "send only input", then your game logic needs to be deterministic. However, the opposite is not true: i.e. if your game logic is deterministic, you are not limited to the "send input" synchronization model. Many games, which uses "send state" model, make heavy use of deterministic.

## What is Deterministic

When a game logic is deterministic, it means given the same input sequence, it will produce the same game state on all machines. If you play the game again with exact same input, you will get the exact same game result. Deterministic is generally a good thing: it enables lots of possibilities that cannot be achieved with indeterministic logic. Then why aren't all all game logic deterministic? Because **achieving deterministic is hard.**

## Deterministic is Hard

Here are some common challenges to implement a deterministic game logic:

- **Floating point:** the same floating point calculation on different machines will give a different result. So to make sure the game logic is deterministic, you have to avoid floating point. But it is not that easy: floating point is such a fundamental numeric type that is everywhere in your game engine, middleware and libraries. For calculations in the logic, usually a fixed point library can be used.
- **Deterministic random:** this is relatively easy. When the game session starts, every player should get the same random seed, which will make sure all of their randomness is deterministic.
- **Ordered data structures:** for example, in most languages, the order of items in a hash (or map or dictionary) is not guaranteed. So looping through hash should be carefully reviewed and prefer something like `SortedDictionary`. This is quite subtle and difficult to trace.
- **Execution order:** for lots of game engines, the update callback order is non-deterministic. For example, in Unity, you should spread the game logic in different Monobehaviors as their update order is not guaranteed. Also Unity coroutine should be avoided in general.
- **Physics:** most physics engines are not deterministic (because of floating point or their internal architecture). However, that does not mean that you cannot use them at all. You can only use indeterministic physics engine for "display". For example. If you make a 3D RTS game, and you have your deterministic collision detection in the game logic based on x and y axis only (e.g. there's no flying unit). Then you are free to use physics engine on the z axis (e.g. climbing up a slope)
- **Separation of logic and view:** your logic should in general avoid relying on view layer (e.g. Animation). For example, when implementing a player ability system, one way is to change state to `USING_ABILITY` when the ability is used, and play the animation. When the animation finishes, add a callback to change the state to `IDLE`. However, this means your logic relies on animation system, which is in general indeterministic. So you have to implement something like this: when a player uses an ability, change the state to `USING_ABILITY` and tell the animation system to play animation. After 10 logic ticks, change the state to `IDLE`, regardless of animation system's state. Also it's very common for logic to run at *different* frequency from rendering frames. Many games run logic tick at 10Hz (or 10FPS) and rendering tick at 60Hz.

## Why Deterministic Then?

As I've mentioned before, if you want to use "send input only" model for synchronization, then your game logic needs to be deterministic, otherwise it will cause desync, i.e. different players in the same multiplayer game will see different gameplays. Even if you use "send state" model, if you can make your game logic deterministic, there are benefits as well. For example, client prediction and rollback, which is usually used to deal with network latency, become more robust and the gameplay is less glitchy. I will go into details later.

## Tracing and Debugging

Tracing indeterministic (desync) is like tracing memory leaks, but worse. At least for memory leaks, there're memory profilers and other tools to help you with that. To trace your deterministic logic, you pretty much have to roll your own tool.

One particular helpful tool that is worth investing in is a logic frame dump, i.e. you dump the game state frame by frame. Then you can compare dumps from different players or even re-run your logic to produce a dump as a benchmark. You can even have some automated tests that re-run the same game inputs several times and assert that all the frame dumps should be exactly the same.

Because desync is hard to reproduce, it is also useful to know how frequent it happens in the real game sessions. You can produce a logic frame dump for every gameplay and send it to server, however, your players will probably complain about the huge network data usage. Instead, consider produce a checksum. And if producing a checksum each frame is too expensive for CPU, you can even sample frames for checksum. In this way, you can have an idea of how bad the desync is in the wild and determine the priority of fixing them.