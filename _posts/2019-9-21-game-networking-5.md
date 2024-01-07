---
layout: post
title: "Game Networking Demystified, Part V: Interpolation and Rollback"
---

Part I: [State vs. Input](https://ruoyusun.com/2019/03/28/game-networking-1.html)  
Part II: [Deterministic](https://ruoyusun.com/2019/03/29/game-networking-2.html)  
Part III: [Lockstep](https://ruoyusun.com/2019/04/06/game-networking-3.html)  
Part IV: [Server and Network Topology](https://ruoyusun.com/2019/04/07/game-networking-4.html)  
Part V: [**Interpolation and Rollback**](https://ruoyusun.com/2019/09/21/game-networking-5.html)  
Part VI: [Game Genres and FAQ](https://ruoyusun.com/2019/09/30/game-networking-6.html)  

*A significant part of this article has been rewritten in Jan 2024 to add discussion on rollback networking*

## Hiding Network Latency

Even though network bandwidth has improved a lot in recent years, the speed of light has remained constant. Therefore, the network latency has remained a problem. Just like real-time rendering is about **"cheating as much as possible"**, netcode latency mitigation follows the same principle. The simplest way is to ignore the latency and focus on hiding it using "visual tricks". In [Part III](https://ruoyusun.com/2019/04/06/game-networking-3.html), we talked about using animation to hide latency.

However, the most commonly perceived latency is about player movement. The easiest trick is to allow the player's visual position to deviate a bit from the logic position. When a movement input is received, the player's visual position is moved immediately while the logic position is moved when it is confirmed from the server. For games with relatively slow movement speed and do not require very accurate and fair collision detection (i.e. not esports), this is often good enough.

The beauty of this trick is that it is very simple: it adds very little complexity and does not add more server computation. It's the low-hanging fruit for mitigating network latency.

## Client Interpolation

For fast-paced esports games, the above trick would not work very well:

1. The faster the player moves, the further the visual and logic position will deviate
2. Collision detection is not accurate (which is a deal breaker for esports shooters)

Client interpolation is a technique to address this problem. The core idea is to show the local player at present and the remote players in the past. When the local players' input is received, it is immediately reflected. For remote players, the position is updated when it is received from the server (thus is in the past).

The downside of client interpolation is collision detection (hit scan) becoming very complex. Because every player's view of the game is different, the server needs to reconstruct each player's view when checking collision. For example, when deciding whether player A has shot player B, the server needs to rewind to player A's view of the game: i.e. with player A at T and player B at T-3 (assuming latency is 3).

This also introduces a potential problem that in player B's view (with player B at T and player A at T-3), the player might feel the shot is missed. In practice, this is less of an issue, because it's much easier for the player to judge whether he/she has shot the target than to judge whether he/she was shot. Another downside of client interpolation is that melee collision detection (as opposed to hit scan) becomes very tricky. That's why you don't see fighting games using this approach - in fact, this technique is most commonly used in competitive shooters.

## Rollback

Another commonly used (and often featured in online discussions) technique is rollback. The core idea is this: we simply predict remote players' input and use that to advance the game state forward without waiting for server confirmation. If our prediction turns out to be wrong, we roll back to the last confirmed game state and make new predictions from there.

The hardest part of rollback networking is **not** the netcode - in fact the netcode implementation is very simple: we save a copy of each predicted frame. And when the server frame arrives (T-3, assuming 3 is the latency), we compare it to our prediction (T'-3), usually via checksum. If it is different, we simply take the server state (T-3), replace our old prediction (T'-3), and make a new prediction (T-2, T-1, T, based on T-3).

The trickiest part is the visual: rollback netcode requires the game's visual to be able to relatively seamlessly transit from one state to another, i.e. `View = f(state)`. In practice, this is very hard: think about all the view states in the game: animations, lighting, physics, UI - asking all these subsystems to rewind to a new state is not an easy job, especially if the game engine is not designed for this (and most game engine is not). 

And even if the game's view can transit from A to B, there will inevitably be visual glitches. For example, if we have predicted that object A has been destroyed but it hasn't. Then the player will see the "dead object come back to life". There are techniques to mitigate this (for example, only destroying an object after receiving confirmation from the server) but they are very hard to get right.

Another obvious problem is about predicting remote input. In general, the prediction accuracy drops as the number of players increases, to the point that prediction is rarely correct, i.e. rollback will almost always happen. This might not be as bad as you think. If the prediction window is small and misprediction only results in a small discrepancy, then correction (i.e. rollback and re-predict) is barely noticeable. In practice, the frequency of misprediction has a smaller impact compared to the prediction window, i.e. increasing the prediction window will result in more noticeable visual glitches (of course this is very game-dependent). One way to address this is to add some input delay: this decreases the prediction window and can help prevent some of the most nasty visual glitches, at the cost of reducing the game's responsiveness.

## Recommended Readings

Gabriel Gambetta has [a series of articles](https://www.gabrielgambetta.com/client-server-game-architecture.html) on client prediction for Client/Server architecture. Valve’s [Source Multiplayer Networking](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking) is also a good read.

Discuss on [Reddit](https://www.reddit.com/r/gamedev/comments/d7frze/game_networking_demystified_part_v_client/)