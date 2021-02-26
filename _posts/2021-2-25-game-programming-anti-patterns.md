---
layout: post
title: Game Programming Anti Patterns
---

I have written quite some code for games in different settings: commercial games/engines, my own [games](https://fishpondstudio.com/)/frameworks and some consulting work (mostly netcode related). Although I'd like to work with clean code, oftentimes I end up working with spaghetti - even for my own games. There are several reasons behind this which I might talk about later. But for this article, I'd like to document some of the common anti patterns in game programming I've seen (and probably have written myself as well) - hopefully it will make you smile.

## Once

Imagine one day before the game release, you have a bug report that your game crashes. You trace down to a method that should only be called once. But it is called more than once!

```javascript
function shouldOnlyCallOnce() {
    // ...
}
```

It's not obvious why. Maybe the method is invoked dynamically (or via reflection), or maybe it's invoked with some weird even mechanism, or even coming from the engine (you don't want to mess with engine!). But there's no time to investigate the root cause as your producer needs you to fix it within an hour. So what do you do? You write something like this.

```javascript
// Remove after finding the root cause for BUG-2 (the bug was created 5 years ago)
let called = false;
function shouldOnlyCallOnce() {
    if (called) return;
    called = true;
    // ...
}
```

This "hack" has been so widely used, some people even have a helper for this to make it look nicer.

```javascript
// Return a function that is func but can only be called once
function makeOnce(func);
```

## Keep Trying

In an `update` loop, you'd expect all its dependencies should be ready. But apparently it's not the case. The object lifecycle and script execution order is already a delicate mess - changing them will cause 100 other things to break. So what's the least disruptive fix?

```javascript
function update(dt) {
    if (this.audio != null) {
        this.audio.play();
    }
    if (this.health != null) {
        this.health.takeDamage();
    }
    // With C# or TypeScript, you can even do something like this
    // this.health?.takeDamage();
}
```

Of course with the fix, the player will miss some damage and maybe some audio effect - but they probably won't realize it anyway. This doesn't look *that bad*, until you combine it with once pattern

```javascript
let damageTaken = false;
function update(dt) {
    if (damageTaken) return;
    
    if (this.health != null) {
        this.health.takeDamage();
        damageTaken = true;
    }
}
```

And you can pretty much say goodbye if you want to use client deterministic net code. An ultimate evolution of this is our next pattern.

## Boolean Execution Order Guard

```javascript
let damageTaken = false;
function takeDamage() {
    damageTaken = true;
    // ...
}

let animationPlayed = false;
function playAnimation() {
    animationPlayed = true;
}

let audioPlayed = false;
function playAudio() {
    audioPlayed = true;
}

function update(dt) {
    if (!damageTaken && !animationPlayer && !audioPlayed) {
        takeDamage();
    }
    if (damageTaken && !animationPlayed && !audioPlayed) {
        playAnimation();
    }
    if (damageTaken && animationPlayer && !audioPlayed) {
        playAudio();
    }
}
```

You would think knowing the order your code is executed is quite basic. Yet after spending countless of hours debugging weird gameplay bugs, I only wish it were true!

## Way Too Much

Oftentimes you find something in your codebase that look like this

```javascript
function walk(isRunning, isFlying, isTeleporting) {
    if (isRunning) {
        // ...
    }
    if (isFlying) {
        // ...
    }
    if (isTeleporting) {
        // ...
    }
}
```

This is not just in games but it happens more often in games. Most likely you start with a simple `walk` function and then a game designer says "what if we allow characters to run?". You add a `isRunning` flag as a quick hack to test "if the game feels right" and people are happy but you never have enough time to properly refactor it. And later "flying ability" is added and of course for paying players, they can "teleport".  Now your `walk` function is used everywhere and refactoring it will break 100 other things.

## Inconsistent Magic Number

Magic number in your code is bad. Inconsistent convention is also bad. What about combining these two evils?

```javascript
function findEnemy(hp: number) {
    if (hp === -1) {
        // -1 means boss
        return boss;
    }
    if (hp === 0) {
        // 0 means return full health enemies
        return enemy.filter((e) => e.hp === e.maxhp);
    }
    return enemy.filter((e) => e.hp >= hp);
}

function findTeammate(hp: number) {
    if (hp === -1) {
        // -1 means all teammates
        return teammates;
    }
    if (hp === 0) {
        // 0 means myself
        return teammates.filter((t) => t.id === me.id);
    }
    // That also will include myself
    return teammates.filter((e) => e.hp >= hp);
}
```

This is somehow related to the "way too much" pattern. But at least that has proper parameters!

## Hidden Cost

Your game has framerate issue so you look at your code

```javascript
function update(dt) {
    this.health += 1;
}
```

That cannot be slow, you wonder. But then you look a bit deeper

```javascript
set health(value) {
    this._health = value;
    const gamesave = this.serialize();
    File.writeToSave(gamesave);
    // and 100 other expensive operations
    // and 100 other big memory allocations
}
```

Aha! This is a common trap to fall into especially you use third party libraries where you don't even have the source code. Having a property getter/setter (like in ES6 or C#) makes it even less obvious. See Unity's infamous [Camera.main](https://blogs.unity3d.com/2020/09/21/new-performance-improvements-in-unity-2020-2/) trap (which has been fixed finally)

## Speculative Optimization

You are asked to slightly change a small feature in your game that only appears once. You locate the code, which looks like this.

```javascript
// We accept rewards array and fill it as output, this saves one allocation
function finalBossKilled(rewards: number[]) {
    const i = 0;
    const arrLength = config.rewards;
    // clear the output array
    rewards.length = 0;
    // We use while loop because its faster!
    while (i < arrLength) {
        rewards.push(config.rewards[i] * 10);
        i++;
    }
}
```

You realize that this is essentially  a one-liner like `config.rewards.map((r) => r * 10)` but with micro optimizations. This function is written as if it were called in an `update` loop. In fact, it's only called once, in a non-performance critical context.

There are usually two common pathways that the code end up like this. Some programmers are obsessed with performance and tend to jump straight into micro optimizations for *everything*. This is not necessarily bad - faster code is in general better than slower code, until you realize the game code is full of them - and hardly maintainable.

The other possible reason is that someone is told "boss flight has framerate drop". The programmer jump right into the code looking for micro optimizations without any profiling. So your code ends up with 100 more micro optimizations and is hardly readable, yet the performance didn't improve that much. The three most important things when it comes to optimization: **profile, profile, profile!**

## Hidden O(N²)

O(N²) is the most common cause for algorithmic performance issues - it's fast enough to get into your game but slow enough to drag the game down when the size reaches couple of hundreds, especially if you are doing it in an `update` loop.

The problem is some of the O(N²) issues are not that obvious, especially if you use a third party library (which makes profiling hard unless you have source code access). When working on adding hexagon grids to [Industry Idle](https://industryidle.com/), I use a library that helps with grid calculations. For every logic tick, I need check all the neighbors of each tile - I assume this is O(N).

The game runs pretty well with a couple of tiles but quickly blows up when I load a medium-sized base. After some profiling, I realize the [neighborsOf](https://github.com/flauwekeul/honeycomb/blob/a1c3606c0763d3e3837c9178db8e37c3626fb69d/src/grid/prototype.js#L252) function in the library actually calls this [get](https://github.com/flauwekeul/honeycomb/blob/a1c3606c0763d3e3837c9178db8e37c3626fb69d/src/grid/prototype.js#L26) function, which calls `array.indexOf()`! So it's actually O(N²).

I am lucky this time because: 1) Chrome has a relatively good profiling/debugging tool; 2) I can inspect the source code of the library relatively easily. Now imagine working on a game project with no debugging tool, no profiler and you have bunch of magic `lib.so` binary in your projects.

Again I need to stress that always profile before doing any optimization. Because in this case, no matter how many `array.foreach()` I replace with `while` loop, I won't be able to fix the real problem, which is `array.indexOf()` in a seemingly harmless function call.

## Misused Events

Gameplay code uses events extensively. But not all usage is justified and misused events can make the code really hard to follow (and debug). You are looking at the following code:

```javascript
function takeDamage(damage) {
    this.player.health -= damage;
    this.trigger("PlayerTakeDamage");
    this.trigger("PlayerHPChanged", player.health);
    if (this.player.health <= 0) {
        this.trigger("PlayerDead");
    }
}

function recover(hp) {
    this.player.health += hp;
    this.trigger("PlayerHPChanged", player.health);
    if (this.player.health > MAX_HEALTH) {
        this.player.health = MAX_HEALTH;
        this.trigger("PlayerHealthFull");
    }
    this.trigger("RecoverUsed");
}
```

This code probably starts with no events at all. But then an event is added because UI needs to listen to the change. And then another one is added because visual effects need to be played with a slightly different condition. And then another one.

You probably want to implement proper observable for `Player`, or at least try to consolidate some of the events and move them into `Player`. But it will probably cause too much disruption since 100 other things are listening to them. Or maybe just add the new event in `Player`? It would be inconsistent - since now the logic is scattered in two places. What do you do? You add a new event in this already "way too much" method.

## Conclusion

This is only a small faction of common anti-patterns that I've seen in game code. And I haven't included *net code* and *UI code* - they are usually the ugliest part of the game code. Game development and software development, while both involve writing code, are fundamentally different business. Between a fun game with bad code and a boring game with clean code, I think most people would prefer former (of course most people would prefer fun game with clean code, but that rarely exists, more on this in another article).