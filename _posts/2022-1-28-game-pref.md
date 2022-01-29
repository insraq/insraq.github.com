---
layout: post
title: Practical Game Performance Optimization - A Real Life Example From Industry Idle
---

Game performance is hard, maybe not as hard as making a game [deterministic](https://ruoyusun.com/2019/03/29/game-networking-2.html#deterministic-is-hard). Both require *disciplines*, which do not come out of the box with programming language and tooling. It's generally a good idea not to worry too much about performance while adding new game features. Because trying to kill two birds with one stone usually leads to bad decisions - over-engineering, premature optimization and the "solution" might not work at all. After all, it's hard to optimize a codebase while new code is constantly being added. The old wisdom of "first you make it work, then you make it fast" applies here.

A Good timing to do performance optimization is after game features are more or less complete. Fortunately, most development platforms have decent profilers - which makes it easy to track down the problem, compared to deterministic issues, which is more like detective work.

## Background And Where Things Stand

[Industry Idle](https://industryidle.com/), if you haven't played before, is essentially an economic simulator. When the economy gets bigger, the simulation will take more time to run. To make things worse, the game also renders all the details on screen as well.

![Game Screen Capture](https://i.ibb.co/bNy5Bwh/ezgif-3-6148af3f77.gif)

In the above screen capture, the tiny "dots" represent real-time resource movement - for example, after an iron mine has mined iron ore which is then sent to a steel factory to produce steel, a dot will move from iron mine to steel factory. Now when the economy and the supply chain get more and more complex, an overwhelming amount of dots will be flying on the screen, kind of like a swarm. It is cool to look at but it will kill the game's FPS.

In previous versions, a workaround is added allowing people to turn off the rendering of those dots. This significantly increases the performance, albeit the game looks much less cool. I'd like to optimize the code so that people with huge economic stimulation can still turn on the dots and have reasonable performance.

## Three Most Important Things For Performance Optimization

The answer to the above question is **profile, profile and profile**. Obviously, a game build with an accompanying game state that can reproduce the performance problem needs to be obtained so we have something to profile on.

![Profile](https://i.ibb.co/R05gQQq/image.png)

From the above profiler result, we now know our baseline. Each frame takes around 163ms - this results in about 10FPS. No wonder that player has turned off those dots! The base has around 800 buildings and 30,000 resources (i.e. dots).

## More Aggressive Offscreen Culling

Even though there are 30,000 dots, not all of them are visible in the current viewport. So skip rendering for those dots that are offscreen is a low-hanging fruit, which is already done by my rendering engine (*cocos2d*). However, the game objects (called `Node` in cocos2d) are still created and ticked before we need to track their position. When the viewport changes, for example when the player zooms out or moves the camera, these will be rendered again.

However, the render culling is not enough - ticking the 30,000 game objects is still too much. So in the previous optimization, when a player turns off dots, no game object is added to the scene at all. After all, from the simulation's point of view, it only needs to know that "iron ore will move from iron mine to steel factory in 2.5 seconds" - there's no need to update its position at all. So the simulation basically only needs to do this

```javascript
deductResourceFrom(ironMine, "iron", amount);
scheduleOnce(() => addResourceTo(steelFactory, "iron", amount), 2.5);
```

Much smaller overhead than creating the game objects! This is why when dots are turned off, the performance is *magnitude better*. One option is to apply this optimization for dots that are offscreen. However, there's a compromise - when the viewport is changed, those dots that are culled are not easily recoverable. But I think it is a reasonable compromise to make and the player should be given the option to make this compromise. So I've added this as an option.

![Profile](https://i.ibb.co/J7pQSgX/image.png)

So after this optimization is introduced, the frame time is reduced from 163ms to 59ms. The total number of game objects for those dots is reduced from 30,000 to 7,000.

## A Faster Game Object Pool

![Profile](https://i.ibb.co/HVfqtkk/image.png)

I have briefly marked what each big chunk of code is doing. Each frame starts with the "logic" tick - this is an earlier optimization. The game's logic tick is not 60FPS - it is much lower than that. However, the logic tick is quite heavy and causes frame rate stutters. The solution is to spread the work into several frames instead of doing it in one frame. This helps a lot with random frame rate drop but doesn't really reduce the total processing time.

What stands out from the flame graph is the "process scene tree" part - because I didn't expect it to take that long at all! Digging a little deeper, it seems that something keeps "dirtying" the scene tree and causing it to calculate the rendering order. I probably should mention that the dots are already using an [object pool](https://en.wikipedia.org/wiki/Object_pool_pattern) implementation from cocos2d.

However, after reviewing the code, I realize that when an object is returned to the pool, it is removed from its current parent, which makes sense for a general object pool design. However, in our case, this is totally unnecessary! We can implement our own object pool that is much more efficient - **because we can make assumptions that a general game engine cannot**. After implementing our own object pool, we manage to get rid of the "process scene tree" part completely for most frames and the frame time is further reduced to 47ms.

![Profile](https://i.ibb.co/pW0Ns8N/image.png)
*(I should also mention that I didn't just pick a random frame to compare. I actually calculated the average frame time and pick a frame that is close to that time for illustration)*

## A Faster Scheduler

As mentioned above, the resource movement is simulated with cocos2d's `Scheduler`, which allows the game to schedule some code to run in the future, kind of like `setTimeout`. Its performance is usually good. However, because of the sheer number of dots, its overhead starts to show up. The scheduler's usage also makes it a bit harder to profile since the code path is harder to trace. It's probably easier to see if we turn off dots.

![Profile](https://i.ibb.co/DMsyWyd/image.png)

*(It's quite hard to take two screenshots of the flame graph with the same scale so I have to manually shrink the second graph in MSPaint)*

To optimize this, instead of scheduling a function call, we simply add the information to a `Map` - the arrival time, resource, amount, and target building. Then in the `tickDots` method, we loop through the map and if a resource has arrived, we remove it from the `Map` and add the resource to the target building. This reduces the frame time from barely under 16.7ms to a comfortable 11ms.

Again this optimization is effective because we can make assumptions about our game - and the engine cannot. Also one could argue the use of `Scheduler` is not a good idea in the first place and I would agree. However, it is the quickest way to get the simulation up and running and we can leave the optimization to a later time.

## Tween Faster

Now let's address the elephant in the room - calculating the position and rendering. This was done by a little [Tween](https://docs.cocos.com/creator/2.4/api/en/classes/Tween.html) helper provided by cocos2d. Again we can write our own code to replace that. As a matter of fact, since we have the `tickDots` already, we could do the work there. We need to add `from` and `to` positions. Another optimization we can do here is to check whether the position is in viewport. If not, we don't need to set the position at all and we can avoid dirtying the game object's transform. The result is as follows.

![Profile](https://i.ibb.co/rsYLPFx/image.png)

The frame time is reduced from 47ms to 24ms, not quite 60FPS, but at least is comfortably 30FPS. This reduction comes from the `Scheduler` and `Tween` change together. The main reason is that our own code can make optimization assumptions and has less overhead compared to the engine code.

Now one could also argue that the new code is probably more CPU cache friendly - since we are looping through a "continuous chunk of data". However, I cannot confirm this - I don't have direct knowledge of V8's internal to tell how would this work at the machine code level. Also, JavaScript is a high-level language that is designed to abstract away the low-level details like memory layout - so the optimization should normally leave that out of consideration - and leave it to the compiler. This is one of the reasons I haven't adopted ECS for this game. Two significant benefits claimed by ECS advocates are ease of code organization and performance. The first one is arguable and varies a lot by gameplay. The second one is dubious in JavaScript world. Also, the rendering time is slightly reduced, mainly due to the fact that we avoid dirtying dots that are outside of the viewport.

## Background Mode

Lots of players leave the game running in the background while doing other tasks. When the game is running in the background, rendering can be completely skipped. This would significantly reduce CPU/GPU usage. While I'd like to make this automatic, `document.visibilityState` API is really flaky and works inconsistently in Electron. So I've made a manual option instead. Another problem is that cocos2d doesn't provide a built-in to turn off the renderer. I cannot turn off the engine completely as some of the APIs (e.g. `Scheduler`) are needed by the game's simulation. Luckily in JavaScript, we can monkey patch. After digging through the source code, I figure out that if I simply patch `cc.renderer.render` method to a no-op, rendering will be skipped while keeping the rest of the engine running. The result is a huge reduction of CPU/GPU usage, as one would expect:

![Resource](https://i.ibb.co/P5H6D2Q/image.png)

*(The above is from my newly built PC running Ryzen 5700G with integrated GPU. I cannot get a discrete GPU at a reasonable price)*

That's all the optimization I've done in [this iteration](https://steamcommunity.com/games/1574000/announcements/detail/4331901783670378108). Even though all examples are from my game with a relatively non-mainstream tech stack, the principles can be applied to elsewhere as well.