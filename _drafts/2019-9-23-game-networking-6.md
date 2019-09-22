---
layout: post
title: "Game Networking Demystified, Part VI: Game Genres and FAQ"
---

Part I: [State vs. Input](https://ruoyusun.com/2019/03/28/game-networking-1.html)  
Part II: [Deterministic](https://ruoyusun.com/2019/03/29/game-networking-2.html)  
Part III: [Lockstep](https://ruoyusun.com/2019/04/06/game-networking-3.html)  
Part IV: [Server and Network Topology](https://ruoyusun.com/2019/04/07/game-networking-4.html)  
Part V: [**Client Prediction**](https://ruoyusun.com/2019/09/21/game-networking-5.html)  
Part VI: Game Genres and FAQ  

There’s no silver bullet when it comes to game networking. The solution depends a lot on game genres and game play design.

## RTS

A player usually controls lots of units in RTS. Therefore, sending state is almost impossible since the game state, which consists of all the units, is simply too big. Almost all old school RTS adopts “deterministic lockstep” model. However, lockstep might be troublesome on mobile network: since any spike will cause the gameplay to lag. Fortunately, old school RTS is not really a thing on mobile: controlling lots of unit on a small mobile touchscreen isn’t really viable.

Most modern “lockstep” model, especially on mobile, is not strictly lockstep anymore. Some uses “client prediction and rollback” model. Some uses “server coordinated timed step” instead of “lockstep”, as we [previously discussed](https://ruoyusun.com/2019/04/06/game-networking-3.html#spikes).

## MOBA

This is where it gets interesting. Both “sending input” and “sending state” model can work. And we do see successful games using either models:

- **Dota 1** is a Warcraft 3 custom map, which inherits “sending input” model. And Warcraft 3, as an RTS title, uses deterministic lockstep.
- **Dota 2** uses Valve’s own Source 2 engine and inherits the “sending state” model (aka. Client/Server).
- **Heroes of the Storm** uses the same engine as Starcraft 2 and inherits the “sending input” model.
- Tencent’s mobile MOBA title **Arena of Valor** uses Unity as the engine and adopts “sending input” model. Even though it is completely viable to use “sending state” model as well. My guess is that “sending input” model requires lower bandwidth, which works better on mobile network.

One way to figure out whether a game uses “sending input” or “sending state” is to through reconnection. If you can almost instantaneously reconnect to a game, most likely the game is using “sending state” model since the client only needs to fetch the latest game state snapshot from server. For “sending input”, the server needs to send all the missing input to the client and client needs to *re-simulate* the game, which can take a while.

## FPS

FPS games usually use “sending state” model because:

- Game state is usually small enough to be sent
- Running important game logic (bullet hit) on the server can reduce cheating
- Most FPS features heavy physics-based interaction and it is in general quite difficult to make physics deterministic

Fast-paced multiplayer FPS is hard to implement. To guarantee a smooth and fair game play, [lots of techniques and tradeoffs are needed](https://ruoyusun.com/2019/09/21/game-networking-5.html). It would be even more tricky if the map is big and there are lots of players, for example, Battle Royale. In this case, some culling algorithm (e.g. distance-based) needs to be implemented on the server so that the client only gets what it needs to know.

## MMO

