---
layout: post
title: "Game Networking Demystified, Part III: Lockstep"
---

Part I: [State vs. Input](https://ruoyusun.com/2019/03/28/game-networking-1.html)
Part II: [Deterministic](https://ruoyusun.com/2019/03/29/game-networking-2.html)
Part III: [Lockstep](https://ruoyusun.com/2019/04/06/game-networking-3.html)
Part IV: Server and Network Topology

The previous post discusses "[deterministic](https://ruoyusun.com/2019/03/29/game-networking-2.html)", now let's talk about its close sibling: lockstep.

## What is Lockstep?

In essence, lockstep means game logic will only advance when inputs from all clients are collected, hence guarantees all clients see the *same* game state.

This makes game logic resembles turn-based games. Imagine a turn-based game where every player has to press "End Turn" button for the game to enter into the next turn. Now, imagine if each turn lasts 1/60s (or 1/10s, to be realistic) - this is lockstep.

## Multiplayer in LAN

In the early days of multiplayer games, the majority of the network is LAN-based, which has relatively low and stable latency (i.e. no spikes). This makes it suitable for lockstep model, we can schedule all the inputs received in turn N to be executed in turn N+2, and expect the network round trip to finish in N+1. If our game logic runs every 100ms, then this is a very reasonable expectation in LAN.

The downside is, if **one player's** network cannot finish within deadline, then **all the players'** game will pause and wait for that slow player. So everyone's experience pretty much depends on the slowest player.

## Lockstep and "State vs. Input"

*Sending input + lockstep + deterministic logic* is a very popular combination. However, that does not mean that we cannot use lockstep in "sending state" model (commonly referred to as Client/Server model, but I will avoid this term as discussed in previous post). In fact, some games do that: the authoritative node will only send the game state once all inputs have been received and processed. One of the benefit is "fairness" - all players are on an equal footing.

## Latency

When using lockstep on the Internet, the easiest approach is to increase the input processing delay: i.e. inputs received in turn N will be scheduled in N+6. However, this will makes game feels "laggy". There are some tricks to hide via some presentation layer tricks. For example, when a player casts an ability, add a *cast animation* before the ability is casted. The animation is played instantly but the actual damage (i.e. game state change) is executed at a later tick. We can even dynamically adjust the delay based on network latency. So players with bad network will send the message earlier to compensate for that - and all players get the same amount of delay.

The idea is to use presentation layer (visual effects, particles, animations) to provide a direct "feedback" to player actions while the real game state is changed at a later time. However, this is not always feasible, depending on the game genre. Also, this will not work well if the network has random spikes, which is rather common with mobile Internet.

## Spikes

Visual tricks cannot help with spikes. Because lockstep will spread one player's spike to every player. This is especially troublesome for mobile games. One solution is to set a *timeout*. Instead of waiting for all players' input before ticking forward, the authoritative node (or the authoritative tick coordinator) will tick forward at a set timeout (e.g. 100ms). If a player's input for a turn is missing, the game logic basically treat it as "no input for this turn". In this case, the lagging player will have a lagging experience and everyone else is unaffected.

The tick coordinator for "sending input" model is usually the relay server. And for "sending state" mode, it is the authoritative node.