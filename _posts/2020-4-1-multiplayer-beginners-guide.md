---
layout: post
title: Beginner's Guide to Making a Multiplayer Game
---

Ever since I have written [a series articles on game networking](https://ruoyusun.com/2019/03/28/game-networking-1.html), I have received quite a few emails asking me more or less the same question: **I want to make a multiplayer game, where do I start?** This prompts me to write a beginner's guide to making a multiplayer game, to answer this exact question.

## Do I Need Multiplayer From Day 1?

Here are three common wisdoms regarding game net code:

1. It is **easy** for a multiplayer game to support "single player" mode. It is **hard** to add multiplayer support to a single player game.
2. The difficulty and complexity of making a multiplayer game is orders of magnitude higher than making a single player game.
3. If you make a single player game and decide to change to multiplayer half way, you are pretty much starting from scratch: the single player game code will not save you much time.

**In short: if your core game play involves multiplayer (i.e. your game will not be shippable and complete without multiplayer), you should start with multiplayer code.**

## Shall I Use My Engine's Net Code?

Yes. At least you should explore that option first. However, keep in mind that the game engine's net code module might or might not fit into your game. If you are experienced (then you probably can skip this article), you might be able to reach a conclusion by looking at the design choices of the net code. If you are a beginner, a good way is to get some small gameplay up and running with the engine's net code module.

Most engine net code has several layers. They usually have a low level API that does network connection and transport and a high level API that synchronize the game state. Even if the latter does not fit, you will most likely still be able to use the low level API. 

At the end of the day, you should utilize game engine as much as you can so that you will focus on the gameplay code.

## Separating and Serialize Game State

If you have decided to make a multiplayer game, you should start with abstracting game state from your game code. You should have the game state as a separate data layer and your game code reading from it. For example, in Unity, instead of scattering your game state variables among dozens of `MonoBehaviour` scripts, put them in one object would be a good start.

Multiplayer game code is essentially trying to replicate the game state to all connected clients on network. And in order to do this, you need to be able to serialize game state and restore it. So carefully design your game state data structures so that it is easy to serialize and restore. Also utilizing a [serialization library](https://google.github.io/flatbuffers/) will give you a head start.

## Choosing a Synchronization Model

There are two major ways to synchronize game state: by sending game state itself or sending only inputs. I have discussed about this in depth in [this article](https://ruoyusun.com/2019/03/28/game-networking-1.html). You should make the decision on which model to use as it will likely to determine how you write your game code. Also once you have made a decision, it would be costly to change it later in development.

One of the important decision regarding this is whether you want to achieve [determinism](https://ruoyusun.com/2019/03/29/game-networking-2.html). As a rule of thumb, the more deterministic your game logic, the less data you need to synchronize. However, fully deterministic game logic is hard, some times almost impossible. And depending on your game genre, you can use a mix of "sending state" and "sending input", but that requires careful implementation.

## Gradually Introduce Complexity

It is relatively easy to write multiplayer net code, it is much harder to write **good** multiplayer net code. The difference is that a **good** multiplayer game has lots of [latency mitigation techniques](https://ruoyusun.com/2019/09/21/game-networking-5.html) (e.g. client prediction and reconciliation) that will drastically complicate your code.

**Don't implement all those techniques from the start, even though you are sure you need it.** Instead, get something up and running with plain and simple net code without any of the tricks. Then you can simulate network latency and gradually introduce those techniques. Oftentimes you will realize that you *don't actually need* lots of those techniques. Also you can mitigate lots of latency issues without complicating your net code (e.g. hide latency with VFX animation or introduce a game design change).

Also at this stage, you can bring in external experts to help you with this and you will likely get more actionable advice. For example, when people ask me to look at a project at very early state, my advice is usually very generic (and you might as well reach the conclusion by reading [my network series](https://ruoyusun.com/2019/03/28/game-networking-1.html) yourself). And when people ask me to look at a specific latency issue, I can come up with more detailed fixes and solutions. Of course, if at this stage some fundamental issues (e.g. synchronization model) are uncovered, it would be much costly to fix.

## Good Enough is Good Enough

When dealing with network latency, make reasonable assumptions and know what is good enough. For example, it is often good enough to ensure a good player experience with ping less than 100ms, assuming you have a game server on each major continent. If a player's latency exceeds that, instead of trying to add some crazy optimizations, accept that it is okay for players to have degraded experience and maybe indicate this on the game UI. Of course the threshold depends on the game genre and live game statistics. Setting up a good monitoring system can help you determine how much you should invest in network optimizations.