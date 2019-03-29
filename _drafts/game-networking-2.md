---
layout: post
title: "Game Networking Demystified, Part II: Deterministic and Lockstep"
---

Deterministic and Lockstep often appears together but really they are two separate concept. In the previous part of the series, I state that if you want to "send only input", then your game logic needs to be deterministic. However, the opposite is not true: i.e. if your game logic is deterministic, you are not limited to the "send input" synchronization model. Many games, which uses "send state" model, make heavy use of deterministic.

## What is Deterministic

When a game logic is deterministic, it means given the same input sequence, it will produce the same game state on all machines. If you play the game again with exact same input, you will get the exact same game result. Deterministic is generally a good thing: it enables lots of possibilities that cannot be achieved with indeterministic logic. Then why aren't all all game logic deterministic? Because **achieving deterministic is hard.**

## Deterministic is Hard

Here are some common challenges to implement a deterministic game logic:

- **Floating point:** the same floating point calculation on different machines will give a different result. So to make sure the game logic is deterministic, you have to avoid floating point. But it is not that easy: floating point is such a fundamental numeric type that is everywhere in your game engine, middleware and libraries. For calculations in the logic, usually a fixed point library can be used.
- **Deterministic random:** this is relatively easy. When the game session starts, every player should get the same random seed, which will make sure all of their randomness is deterministic.
- **Ordered data structures:** for example, in most languages, the order of items in a hash (or map or dictionary) is not guaranteed. So looping through hash should be carefully reviewed and prefer something like `SortedDictionary`. This is quite subtle and difficult to trace.
- **Execution order:** for lots of game engines, the update callback order is non-deterministic. For example, in Unity, you should spread the game logic in different Monobehaviors as their update order is not guaranteed. Also Unity coroutine should be avoided in general.
- **Physics:** most physics engines are not deterministic (because of floating point or their internal architecture). However, that does not mean that you cannot use them at all. You can only use indeterministic physics engine for "display". For example. If you make a 3D RTS game, and you have your deterministic collision detection in the game logic based on x and y axis only (e.g. there's no flying unit). Then you are free to use physics engine on the z axis (e.g. climbing up a slope)

## Why Deterministic Then?

As I've mentioned before, if you want to use "send input only" model for synchronization, then your game logic needs to be deterministic, otherwise it will cause desync, i.e. different players in the same multiplayer game will see different gameplays. Even if you use "send state" model, if you can make your game logic deterministic, there are benefits as well. For example, client prediction and rollback, which is usually used to deal with network latency, become more robust and the gameplay is less glitchy. I will go into details later.

## What is Lockstep

