---
layout: post
title: "Game Networking Demystified, Part VI: Game Genres and FAQ"
---

Part I: [State vs. Input](https://ruoyusun.com/2019/03/28/game-networking-1.html)  
Part II: [Deterministic](https://ruoyusun.com/2019/03/29/game-networking-2.html)  
Part III: [Lockstep](https://ruoyusun.com/2019/04/06/game-networking-3.html)  
Part IV: [Server and Network Topology](https://ruoyusun.com/2019/04/07/game-networking-4.html)  
Part V: [Client Prediction](https://ruoyusun.com/2019/09/21/game-networking-5.html)  
Part VI: [**Game Genres and FAQ**](https://ruoyusun.com/2019/09/30/game-networking-6.html)  

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

One way to figure out whether a game uses “sending input” or “sending state” is to through reconnection. If we can almost instantaneously reconnect to a game, most likely the game is using “sending state” model since the client only needs to fetch the latest game state snapshot from server. For “sending input”, the server needs to send all the missing input to the client and client needs to *re-simulate* the game, which can take a while.

## FPS

FPS games usually use “sending state” model because:

- Game state is usually small enough to be sent
- Running important game logic (bullet hit) on the server can reduce cheating
- Most FPS features heavy physics-based interaction and it is in general quite difficult to make physics deterministic

Fast-paced multiplayer FPS is hard to implement. To guarantee a smooth and fair game play, [lots of techniques and tradeoffs are needed](https://ruoyusun.com/2019/09/21/game-networking-5.html). It would be even more tricky if the map is big and there are lots of players, for example, Battle Royale. In this case, some culling algorithm (e.g. distance-based) needs to be implemented on the server so that the client only gets what it needs to know.

## MMO

MMO games almost always use “sending state” model simply because the game state has to be persisted on the server. Plus, since each player usually controls one character, so “sending input” does not save much bandwidth compared to “sending state”.

The biggest challenge though, is the sheer amount of data needed for game state. Obviously, we need some interest management (i.e. culling). For example, we usually don’t care about players that are very far away from us so the server can exclude those users from state update.

Another challenge is the world size. If the whole map is too big for one server to handle, we will need to divide maps into “tiles” or “chunks” and each server handles game logic for one chunk. We also need another server that forwards the player to the corresponding “tile server” given the player position and potentially pass the user info along when user moves from one tile server to another.

Of course, the gameplay of MMO is very versatile: and we might need a mix if servers for different needs. For example, it is reasonable to have a dedicated chat server that handles only chat. Some games also have a dedicated login server for authentication.

## TCP vs UDP?

In general, UDP is favored over TCP. One of the important reason is that TCP’s packet loss recovery isn’t really designed for games. Most games can handle a certain degree of packet loss. However, TCP, being a reliable protocol, will try to resend the any lost packets and reduce the throughput of the network as a side effect.

For fast paced games, it is always much better to implement a custom pocket loss recovery strategy compared to the default TCP implementation. There are several protocols that provide a reliability layer on top of UDP: [ENet](http://enet.bespin.org/), [RakNet](https://github.com/facebookarchive/RakNet) and [KCP](https://github.com/skywind3000/kcp/blob/master/README.en.md).

However, if our game play is not that fast-paced and can get away with TCP, we can also consider using it instead. Because TCP is so widely used, the whole ecosystem around it is very mature so that might saves lots of development time.

## Server Programming Language?

Servers can be done with different programming languages. However, there are some pros to use the same programming languages as client:

- If server is game logic heavy (e.g. sending state model), we can reuse the code. And in some cases, our game server can be a “headless” version of the client.
- It is easier for client and server programmers to work together with a common programming language.

C++ as a traditional game client programming language, can also be used for game servers. However, as a low-level system programming language without garbage collection, it might be more challenging to work with from server side. C#, which is the Unity scripting language, has a bigger and more mature ecosystem for creating scalable servers, especially with .Net Core.

However, it is completely fine to create game servers using a different language, especially you don’t need to reuse client code or your client language does not have a mature server ecosystem. In this case, the mature ecosystems might save you more time.