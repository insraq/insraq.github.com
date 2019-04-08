---
layout: post
title: "Game Networking Demystified, Part III: Server and Network Topology"
---

Part I: [State vs. Input](https://ruoyusun.com/2019/03/28/game-networking-1.html)  
Part II: [Deterministic](https://ruoyusun.com/2019/03/29/game-networking-2.html)  
Part III: [Lockstep](https://ruoyusun.com/2019/04/06/game-networking-3.html)  
Part IV: [**Server and Network Topology**](https://ruoyusun.com/2019/04/07/game-networking-4.html)  
Part V: Client Prediction  
Part VI: Examples with Different Game Genres  

## Physical Server

Let's first be clear with the terminology. The server referred to in this post means "physical server". Please do not confuse with the commonly used "client/server" term – that describes a game synchronization model and the "server" in "client/server" can actually be one of the game client (i.e. host). We have been using the term "authoritative node" to describe that.

## Peer to Peer vs. Central Server

Whichever synchronization model is chosen (sending state vs. sending input), we can use either network topology. In fact, we can even allow *both*. Normally, games will only choose one and here're some considerations.

|                          | P2P                                                          | Server                                                       |
| ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Cost for "Sending Input" | Almost nothing. However, a server for NAT punch-through might be needed | Relatively inexpensive. The server mostly relays network packets |
| Cost for "Sending state" | Same as above.                                               | Relatively expensive. The server has to run the game logic.  |
| Cheat prevention         | No prevention                                                | Good prevention                                              |
| Network Latency          | Depend on connection among players, which we don't have any control | Depend on connection to the server. We can have regional servers and route players to the nearest one |

## NAT Punch-through Server

Even if we choose peer-to-peer network topology, we might still need a minimal server for NAT punch-through. [NAT punch-through](https://en.wikipedia.org/wiki/Hole_punching_(networking)) is basically a trick to allow players behind routers or firewalls to directly talk to each other. This server is very cheap to run: after players have established direct connection, the server is no longer needed.

## Relay Server

Relay server does exactly what the name suggests. Relay servers deployed across multiple regions can decrease network latency. For "sending input" model, the only server needed is relay server as game logic is calculated on each player's local client. Relay servers can also act as tick coordinator, as discussed in the [previous post](https://ruoyusun.com/2019/04/06/game-networking-3.html).

## Game Logic Server

Game logic server is only needed for "sending state" model. It is the authoritative node that calculates the game state and send it to all players. The easiest way to build the game logic server is to have a "headless" build of the game (i.e. without graphics). However, this can get very expensive to run. Alternatively, the game logic can be written in a way to run in a standalone mode, which might be a bit cheaper to scale.

## Other Servers

Apart from the above three types of servers for multiplayer game, several other types might be needed:

- **Matchmaking server**: as its name suggests, matchmaking server is for matching players and group them into a game session. After players are matched, they can communicate either through a server or using peer-to-peer. And in the peer-to-peer case, matchmaking server can be an extension to the NAT punch-through server.
- **Player identity server**: sometimes we might want to store user data on our server: user id, rank, in-game currency owned, etc. This is usually similar (with HTTP API) regardless of the game genre.