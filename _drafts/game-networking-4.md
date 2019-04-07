---
layout: post
title: "Game Networking Demystified, Part III: Server and Network Topology"
---

Part I: [State vs. Input](https://ruoyusun.com/2019/03/28/game-networking-1.html)  
Part II: [Deterministic](https://ruoyusun.com/2019/03/29/game-networking-2.html)  
Part III: [Lockstep](https://ruoyusun.com/2019/04/06/game-networking-3.html)  
Part IV: **Server and Network Topology**

## Physical Server

Let's first be clear with the terminology. The server referred to in this post means "physical server". Please do not confuse with the commonly used "client/server" term – that describes a game synchronization model and the "server" in "client/server" can actually be one of the game client (i.e. host). We have been using the term "authoritative node" to describe that.

## Peer to Peer vs. Central Server

