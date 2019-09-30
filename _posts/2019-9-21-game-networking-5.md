---
layout: post
title: "Game Networking Demystified, Part V: Client Prediction"
---

Part I: [State vs. Input](https://ruoyusun.com/2019/03/28/game-networking-1.html)  
Part II: [Deterministic](https://ruoyusun.com/2019/03/29/game-networking-2.html)  
Part III: [Lockstep](https://ruoyusun.com/2019/04/06/game-networking-3.html)  
Part IV: [Server and Network Topology](https://ruoyusun.com/2019/04/07/game-networking-4.html)  
Part V: [**Client Prediction**](https://ruoyusun.com/2019/09/21/game-networking-5.html)  
Part VI: [Game Genres and FAQ](https://ruoyusun.com/2019/09/30/game-networking-6.html)  

## Why Client Prediction?

Network has latency and if the player has to wait for the network before his input is reflected on screen, the game will feel “laggy”. In [Part III](https://ruoyusun.com/2019/04/06/game-networking-3.html), we mentioned that we can use visuals to trick players, however, it is not always possible. Therefore client prediction is often used.

However, client prediction is not easy: it can drastically complicate our game logic. So if we can use visual tricks, prefer that instead. Also, client prediction is applicable to both sending input model and sending state model. However, it is more widely used in the sending state model (or Client/Server).

## Prediction

There are two major techniques for client prediction:

**Dead reckoning, or extrapolation** uses historical data to predict “future”. This works well for racing games, as given the speed and direction, it is easy to predict the future position.

**Entity interpolation**, instead of predicting the other players, shows the “past state” of the other players. This is the only way if you cannot produce a meaningful future prediction based on past data (like FPS shooter).

In general, extrapolation is easier to implement, but is oftentimes not possible. The prediction also benefits from a deterministic logic: the more deterministic the logic, the more accurate the prediction.

## Reconciliation and Correction

When making predictions, the current player is always ahead of the server. Therefore, when server confirmation arrives, the confirmed game state is actually in the past. If we directly apply the game state, the current player will be “teleported back”, which is obviously not acceptable. The solution is to keep a buffer of all unconfirmed user input. When server confirmation arrives, we discard all the confirmed input and make new prediction based on unconfirmed ones.

If our new prediction is different from the old prediction (i.e. current player state), this means an prediction error has happened. In this case, since server is usually authoritative, we need to correct the prediction mistake. We can do this in a short time (using interpolation) instead of immediately teleport the player.

## Collision Detection

Client prediction significantly complicates the game logic, especially when server-side collision detection is needed. Usually collection detection is done on the server-side (i.e. authoritative node) to prevent cheating. Because of client prediction, we need to restore the correct timing. There are two sources of correction:

If the current player shoots an enemy, when the information arrives at server, the enemy on the server might have moved to a new position. This is *network latency*.

If we use entity interpolation, the player actually sees enemies in the past. This is *interpolation delay*.

When we do server-side collision detection, we need to rewind all other players’ position by *network latency + interpolation delay*, otherwise the collision detection will not be correct.

## Recommended Readings

Gabriel Gambetta has [a series of articles](https://www.gabrielgambetta.com/client-server-game-architecture.html) on client prediction for Client/Server architecture. Valve’s [Source Multiplayer Networking](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking) is also a good read.

Discuss on [Reddit](https://www.reddit.com/r/gamedev/comments/d7frze/game_networking_demystified_part_v_client/)