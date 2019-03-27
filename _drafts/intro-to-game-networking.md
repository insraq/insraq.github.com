---
layout: post
title: Introduction to Game Networking, Part I
---

Reading discussions on game networking is often confusing. People use different "terms" to describe a game's networking architecture but most of the terms only describe part of the architecture. So in one article, people might describe Starcraft 1 is *"peer-to-peer"*, and in another talk, they might state that Starcraft uses *"lockstep"* and in another discussion, you might see *"deterministic"* mentioned as well.

Game networking is complicated: there's no "golden architecture" and most games uses a mix of different techniques depending on the gameplay and limitations. So in this article, I want to discuss commonly used "terms" in a very straightforward way, and also talk about the design choices behind and the implications, hopefully to help you design your multiplayer game networking.

## State vs. Input

Now this is arguably the first big decision you need to make when designing how your game will work across network. To enable network-based multiplayer, you need to send data among all players to make sure all players have the *same game state*. The question is: what to send?

**You can send game state.** A typical example:

- Player A sends  `MoveTo(1, 2)` to *authoritative node*
- *Authoritative node* receives the request, process it and tell all players that Player A is now at `(1, 2)`
- Player A receives the response and moves to `(1, 2)`. Other players all receive the same information and the Player A on their screen all moves to `(1, 2)`.

I use the term *authoritative node*. Most people will call it "server". Therefore, this is sometimes described as "authoritative servers" or "dumb clients" or "client-server architecture". These descriptions are confusing because if you come from other programming areas (like me), *client* and *server* are not necessary what you would imagine. It's quite common to have the *client* and *server* code to run on the same player, often called *"host"*. Of course, you can run the "server code" on a real server, which is separate from the clients. In that case, the server is basically running a "headless" version of your game.

**You can send game input.** Example:

- Player A sends `MoveTo(1, 2)` to all players.
- Player A receives the acknowledgement from all players and moves to `(1, 2)`. Player A on all other players' screen moves to `(1, 2)`.

Compared to the previous example, the main difference is that instead of sending A's position (i.e. game state), we simply send A's input (move). In the first example, the state calculation is done by **one** source, the authoritative node (i.e. host or server). All other players take it as the ultimate truth. However, in the second example. the state calculation is done by all players **separately**. So to make sure all players have the same game state, your game logic needs to be "deterministic", i.e. given the same input, it should produce the same game state.

This is sometimes referred to as "deterministic lockstep", which is confusing because even though this is often used with "lockstep" technique, it is perfectly okay not to lock step. In fact, a lot of mobile games don't lock step because of mobile network. I will discuss lockstep later.

In this example, we assume a peer-to-peer connection. In fact, most modern games use *relay node* to forward inputs to all players instead of using a peer-to-peer connection. Some people call it "relay server" but I'd rather avoid the term "server". In our first example, the "client-server architecture", we can also use a peer-to-peer connection as well. The "server" in "client-server" means "authoritative node/peer", instead of a real physical server.

Confused? Here's a simple rule: forget about all the client and server confusions. **Just ask the question: what data are sent to keep the game state in sync: is it the game state itself or only the inputs?**