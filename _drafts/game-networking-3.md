---
layout: post
title: "Game Networking Demystified, Part III: Lockstep"
---

The previous post discusses "[deterministic](https://ruoyusun.com/2019/03/29/game-networking-2.html)", now let's talk about its close sibling: lockstep.

## What is Lockstep?

In essence, lockstep means game logic will only advance when inputs from all clients are collected, hence guarantees all clients see the *same* game state.

This makes game logic resembles turn-based games. Imagine a turn-based game where every player has to press "End Turn" button for the game to enter into the next turn. Now, imagine if each turn lasts 1/60s (or 1/10s, to be realistic) - this is lockstep.

## A Brief History

