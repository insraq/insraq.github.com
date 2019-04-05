---
layout: post
title: "Game Networking Demystified, Part I: State vs. Input"
---

Part I: [State vs. Input](https://ruoyusun.com/2019/03/28/game-networking-1.html)
Part II: [Deterministic](https://ruoyusun.com/2019/03/29/game-networking-2.html)
Part III: [Lockstep](https://ruoyusun.com/2019/04/06/game-networking-3.html)
Part IV: Server and Network Topology

Reading discussions on game networking is often confusing. People use different "terms" to describe a game's networking architecture but most of the terms only describe part of the architecture. So in one article, people might describe Starcraft 1 is *"peer-to-peer"*, and in another talk, they might state that Starcraft uses *"lockstep"* and in another discussion, you might see *"deterministic"* mentioned as well.

Game networking is complicated: there's no "golden architecture" and most games uses a mix of different techniques depending on the gameplay and limitations. So in this article, I want to discuss commonly used "terms" in a very straightforward way, and also talk about the design choices behind and the implications, hopefully to help you design your multiplayer game networking.

## State vs. Input

First, let's talk about "State vs. Input". This is arguably the first big decision you need to make when designing how your game will work across network. To enable network-based multiplayer, you need to send data among all players to make sure all players have the *same game state*. The question is: what to send? There are two common approaches and they will significantly shape your game code.

## Sending State

**You can send game state.** A typical example:

- Player A sends  `MoveTo(1, 2)` to *authoritative node*
- *Authoritative node* receives the request, process it and tell all players that Player A is now at `(1, 2)`
- Player A receives the response and moves to `(1, 2)`. Other players all receive the same information and the Player A on their screen all moves to `(1, 2)`.

I use the term *authoritative node*. Most people will call it "server". Therefore, this is sometimes described as "authoritative servers" or "dumb clients" or "client-server architecture". These descriptions are confusing because if you come from other programming areas (like me), *client* and *server* are not necessary what you would imagine. It's quite common to have the *client* and *server* code to run on the same player, often called *"host"*. Of course, you can run the "server code" on a real server, which is separate from the clients. In that case, the server is basically running a "headless" version of your game.

## Sending Input

**You can send game input.** Example:

- Player A sends `MoveTo(1, 2)` to all players.
- Player A receives the acknowledgement from all players and moves to `(1, 2)`. Player A on all other players' screen moves to `(1, 2)`.

Compared to the previous example, the main difference is that instead of sending A's position (i.e. game state), we simply send A's input (move). In the first example, the state calculation is done by **one** source, the authoritative node (i.e. host or server). All other players take it as the ultimate truth. However, in the second example. the state calculation is done by all players **separately**. So to make sure all players have the same game state, your game logic needs to be "deterministic", i.e. given the same input, it should produce the same game state.

This is sometimes referred to as "deterministic lockstep", which is confusing because even though this is often used with "lockstep" technique, it is perfectly okay not to lock step. In fact, a lot of mobile games don't lock step because of mobile network. I will discuss lockstep later.

## The Confusing "Server"

In our second example, we assume a peer-to-peer connection. However, most modern games use *relay node* to forward inputs to all players instead of using a peer-to-peer connection. Some people call it "relay server" but I'd rather avoid the term "server". In fact, In our first example, the "client-server architecture", we can also use a peer-to-peer connection as well. The "server" in "client-server" means "authoritative node", instead of a real physical server.

Confused? Here's a simple rule: forget about all the client and server confusions. This is not about communication model (peer-to-peer, server-client or relay), this is about game state synchronization. **Just ask the question: what data are sent to keep the game state in sync: is it the game state itself or only the inputs?**

## Which One To Use?

As I stated before, the answer to this questions is: it depends. Here's a brief comparison:

|                     | Sending State                                                | Sending Input                                                |
| ------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Network Traffic     | Usually **high**                                             | Usually **low**                                              |
| Reconnect           | Relatively **easy**, the player simply needs to retrieve the latest game state from authoritative node | Relatively **difficult**. You need a relay node to store all the inputs. When the player reconnects, all the inputs will be resent and replayed on the player's side |
| Replay File Size    | Big                                                          | Small                                                        |
| Cheat               | Relatively **hard**, especially your authoritative node is on your physical server | Relatively **easy**.                                         |
| Deterministic Logic | Not necessary                                                | Necessary                                                    |
| Typical Genre       | MMO                                                          | RTS                                                          |

## Cheat and Prevention

Under either approach, a player can hack the game to cheat. I will discuss three common cheats and the ways to prevent them.

### A player hacks HP from 100 to 200

**Sending State:** If the player is not an authoritative node, this is usually not a problem since the next time when game state is synchronized, all the hacked values will be discarded. However, if the player is an authoritative node (i.e. host). it will be a problem. The common solution is to run your authoritative node on your physical server where only you can access it.

**Sending Input:** In this cases, players will have a desynced game play. i.e. The hacking player will have 200 HP on *the local game* but all other players will see 100 HP. There are two common ways to solve this:

- Sample game state checksums to detect "desync" and kick out the desynced player. However you might not want to kick out the player *right away* when there's a desync because it can happen if your game logic is indeterministic (because achieving deterministic is hard, which I will cover later). A practical approach is to only kick out a player when desync exceeds a threshold.
- Re-enact and validate the play session. For this, you need a relay server to record all the inputs and re-enact the play session. In this way, you will get a result without desync and hack. Of course, when re-enacting, the cheater's game play will be very messed up as the input will not make sense any more. But we don't really need to care about the cheater's game experience.

These two approaches are not exclusive. In fact, most games uses a combination.

### A player hacks and removes fog of war

**Sending State:** Again, first you need to put your authoritative node on your physical server. And then your authoritative node needs to be a bit "smarter" and only sends the game state that a player should know. If you have done this, then even if the cheater lifts fog of war, the game state will *not* be revealed.

**Sending Input:** Now this is when things get tricky. Because the game state needs to be calculated locally, the player knows all the game state. There's no good way to prevent cheaters revealing the whole game state. Of course you can add some memory obfuscation or game binary protection to make it harder, but it is only a matter of time (and effort).

### A player hacks and uses auto-aim

In this case, neither of the two model will have a *bulletproof* way to prevent cheating. Because the cheater makes use of the known local game states. Your only hope will be binary protection, memory obfuscation, detecting commonly used plugins (kind of like anti-virus) and behavior detection.

This concludes part I of the series. In the next part, we will talk about *deterministic* and *lockstep*.