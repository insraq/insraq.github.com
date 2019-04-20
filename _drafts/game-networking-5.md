---
layout: post
title: "Game Networking Demystified, Part V: Client Prediction"
---

Part I: [State vs. Input](https://ruoyusun.com/2019/03/28/game-networking-1.html)  
Part II: [Deterministic](https://ruoyusun.com/2019/03/29/game-networking-2.html)  
Part III: [Lockstep](https://ruoyusun.com/2019/04/06/game-networking-3.html)  
Part IV: [Server and Network Topology](https://ruoyusun.com/2019/04/07/game-networking-4.html)  
Part V: **Client Prediction**  
Part VI: Examples with Different Game Genres  

## Recommended Reading

Gabriel Gambetta has [a series of articles](https://www.gabrielgambetta.com/client-server-game-architecture.html) on client prediction for Client/Server architecture. I strongly recommend reading it. In this post, we will briefly go through the concepts and discuss some additional approaches

## Why Client Prediction?

Network has latency and if the player has to wait for the network before his input is reflected on screen, the game will feel “laggy”. In [Part III](https://ruoyusun.com/2019/04/06/game-networking-3.html), we mentioned that we can use visuals to trick players, however, it is not always possible. Therefore client prediction is often used.

However, client prediction is not easy: it can drastically complicate our game logic. So if we can use visual tricks, prefer that instead. Also, client prediction is applicable to both sending input model and sending state model. However, it is more widely used in the sending state model (or Client/Server).

There are big problems regarding client prediction: **prediction and correction**. *We want our prediction to be as accurately as possible and if our prediction happens to be wrong, we want our correction to be as less awkward as possible.*

## Prediction

There are two major techniques for client prediction:

**Dead reckoning** uses historical data to predict “future”. This works well for racing games, as given the speed and direction, it is easy to predict the future position.

**Entity interpolation**, instead of predicting the other players, shows the “past state” of the other players. This is the only way if you cannot produce a meaningful prediction based on past data (like FPS shooter). And when a player shoots another, the server needs to reconstruct the game state with synced time to determine whether it is a shot or miss.

The prediction also benefits from a deterministic logic: the more deterministic the logic, the more accurate the prediction.

## Correction

