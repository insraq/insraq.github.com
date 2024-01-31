---
layout: post
title: "Squeezing Every Bit Of Performance Out Of JavaScript For My Automation Game"
---

Shortly after I released [Industry Idle](https://industryidle.com/), I did a big round of optimization which I [wrote about](https://ruoyusun.com/2022/01/28/game-pref.html). For those of you who've never played Industry Idle, it is a factory automation and economy simulation game. The main performance issue I ran into is actually on the rendering side, which is surprising given its minimalist graphics.

![CivIdle](https://github.com/insraq/insraq.github.com/assets/608221/8a4e9749-326b-4da6-ae96-3a68fec2de9c)

## Why TypeScript /JavsScript Again?

When I decided to work on its spiritual successor [CivIdle](https://www.cividle.com/), I decided to rewrite from scratch. CivIdle takes the core idea of Industry Idle and adopts a historical theme. The game adds several new mechanisms like a tech tree, world wonders, and great people - these all add more complications to the game's simulation.

Since I am rewriting everything, I am also exploring languages other than TypeScript - after all, TypeScript/JavaScript is not my usual first choice if I need to write high-performance code. However, after taking a look at Industry Idle codebase, I've reached the conclusion that 90% of the game's code is UI and the web platform is painfully good at this. Seriously, if you think web frameworks are hard to work with, you should try using a game UI library/middleware. Since the majority of CivIdle code is likely gonna be UI code, I'd like to stay within the web platform for UI, which leaves me with a few choices:

1. Use the web platform (like Industry Idle): use HTML for UI and WebGL for game rendering, and write TypeScript for both;
2. Same as above but use WebAssembly for game logic and rendering - UI will remain in HTML/TypeScript;
3. Write the game logic and rendering in C++, use a middleware ([1](https://ultralig.ht/), [2](https://coherent-labs.com/products/coherent-gameface/)) that supports rendering HTML on top of the 3D context;

I started experimenting with option 3 but quickly gave up because the amount of bridging code needed was huge. All those middlewares provide subpar developer tooling compared to the browser, which makes developing, especially debugging quite painful. And sharing large data (like the game state) safely and efficiently between C++ and JavaScript is not trivial either. Option 2 suffers from similar issues apart from improved tooling when working on UI. That pretty much leaves me with option 1.

## Taking All the Lessons Learned

Industry Idle's performance problem with rendering mostly results from the game engine (Cocos Creator). The engine is designed to be a Unity-like all-in-one game engine: with a visual editor and its own scene graph/prefab system. Even though I only use its rendering, it's not easy to customize its renderer for performance - since this is probably not its designed use case.

For CivIdle, I've chosen [Pixi.JS](https://pixijs.com/), one of the fastest WebGL renderers. It does one thing and does it well - exactly what I need. I briefly toyed with the idea of writing my own WebGL renderer but decided that I probably wouldn't be able to do a better job than Pixi.JS.

With all the learnings from Industry Idle, I started with a higher bar already: objects are pooled whenever possible, textures are optimized and packed into atlases, and draw calls are diligently checked, those tiny dots that are problematic in Industry Idle are implemented using specialized `ParticleContainer`. The only issue I have is the [MSDF](https://github.com/Chlumsky/msdfgen) font rendering consistently breaks batching (due to requiring its own shader) and I've replaced it with the good old bitmap font, albeit sacrificing a bit of the quality.

## Hitting the Performance Wall

Industry Idle's biggest architectural flaw is that the game logic and rendering are tightly coupled, which means I cannot run the game logic without rendering. The offline earning in Industry Idle is done by an approximating calculation. For a hardcore simulation games fan like myself, this feels like cheating - I think it's important for a simulation game to faithfully simulate everything. So ideally offline earning should work by taking the offline time and running the actual simulation for that period.

For CivIdle, the simulation and rendering are completely decoupled to achieve the "simulate everything" design. After I implemented offline earning and ran it on a large map for the first time, my heart sank: it took about 43 seconds to run an hour's simulation, which is about 12ms wall clock time per game second. This means if the player was offline for a day, it would take around 17 minutes for the simulation to catch up! When I first released the offline production feature to the playtesters, I received several bug reports saying the game was "stuck" at loading - in fact it was not stuck, just painfully slow at catching up.

To put it into context, 12ms is still within the 16.67ms frame time budget to achieve a smooth 60 FPS. However, this is not enough to achieve a reasonable offline production performance. The game logic runs in a single thread and cannot easily be parallelized - plus adding multithread to JavaScript requires Web Worker, which is not trivial to set up. In addition, Web Worker would be helpful to unlock the main thread, but it probably won't reduce the overall time. So we need to squeeze as much ingle-thread performance as possible.

## It's Always Allocation!

Before we start to optimize, we need to figure out where are the CPU time spent. Luckily, the browser has really good profiling tools. Running a profile session in Chrome reveals the following result.

![](https://github.com/insraq/insraq.github.com/assets/608221/89a4ed63-0f46-4786-a94c-66717d5d007d)

The profiler shows a single hot function, which is promising - this usually indicates a low-hanging fruit. The function in question is this:

```typescript
public distance(x1: number, y1: number, x2: number, y2: number): number {
   const oc1 = new OffsetCoord(x1, y1);
   const hex1 = OffsetCoord.roffsetToCube(OffsetCoord.ODD, oc1);
   const oc2 = new OffsetCoord(x2, y2);
   const hex2 = OffsetCoord.roffsetToCube(OffsetCoord.ODD, oc2);
   const distance = hex1.distance(hex2);
   return distance;
}
```
It's calculating the distance between two tiles on the hex grid - and it's called a lot. This code is shamelessly copied from the brilliant Red Blob Games' [reference hex grid implementation](https://www.redblobgames.com/grids/hexagons/codegen/output/lib.ts). The actual calculation didn't show up in the profiler, it's the allocation of all the small objects!

In a language that has value types, this is usually not a big deal. These can be allocated on the stack and are generally very fast. In our JavaScript VM (V8), almost everything is allocated on the heap (except for "small integers", which we will get to later), and constructing these objects (setting up the prototype chain) is surprisingly heavy. The solution is to use a static cached copy of these objects, instead of allocating it every time. Also, we need to add a non-allocating version of the API: they either set the result to an object (which is a cached static) that is passed in or mutate one of the objects (which is again a cached static). The new code looks like this:

```typescript
private static _oc1 = new OffsetCoord(0, 0);
private static _oc2 = new OffsetCoord(0, 0);
private static _hex1 = new Hex(0, 0, 0);
private static _hex2 = new Hex(0, 0, 0);

public distance(x1: number, y1: number, x2: number, y2: number): number {
   Grid._oc1.col = x1;
   Grid._oc1.row = y1;
   OffsetCoord.roffsetToCubeNonAlloc(OffsetCoord.ODD, Grid._oc1, Grid._hex1);
   Grid._oc2.col = x2;
   Grid._oc2.row = y2;
   OffsetCoord.roffsetToCubeNonAlloc(OffsetCoord.ODD, Grid._oc2, Grid._hex2);
   const distance = Grid._hex1.distanceNonAlloc(Grid._hex2);
   return distance;
}
```

The new code looks considerably worse, and is not thread-safe - but we are single-threaded anyway. Doing the same profile run: the overall time reduces from 21s to 9s! That's a sweet low-hanging fruit!

![](https://github.com/insraq/insraq.github.com/assets/608221/936b6f25-c730-4306-90e6-5ef664b8c8e1)

## Memorizing And Caching

Looking again through the profile result, several functions stand out to me. They take a decent chunk of the CPU time yet the function only contains some "pure" calculations: they do not rely on external states and they do not have side effects. An example looks as follows (I have simplified the function signature: in the real code base, the `string` has a type, but it's irrelevant here)

```typescript
function getBuildingCost(type: string, level: number): Record<string, number>
```

These are great candidates for memoization, which is basically a fancy way of saying: cache the computation result instead of computing it every time. A generic `memoize` function can look like this

```javascript
function memoize(func) {
   const results = {};
   return (...args) => {
      const argsKey = JSON.stringify(args);
      if (!results[argsKey]) {
         results[argsKey] = func(...args);
      }
      return results[argsKey];
   };
}
```
Using `JSON.stringify(args)` to generate a unique "hash" for the arguments is not terribly efficient. In the case of `getBuildingCost`, we can probably just concatenate the arguments.

Several functions do rely on external states for calculation, but the state does not change within a frame: we can cache the result and clear the cache at the beginning of a new frame.

Add memoization shaves another ~2s from the overall time, not too bad!

## Getting Rid Of Strings!

Unfortunately, our low-hanging fruit has all been picked. At this point, most of the hot spots in the code involve strings:

1. In our previous memoization example, we've used a string key
2. The game uses `{x, y}` to represent a tile and whenever a key is needed, it uses the string form `"x,y"`. Several hot spots involve converting between the two forms

Strings make the code easy to read. Unfortunately, they also make the code slow to run. However, getting rid of them is not always trivial. In the case of representing a tile, we can pack `x` and `y` into an integer:

```typescript
const tile = (x << 16) |  y;
```

Previously we mentioned that almost everything in JavaScript (V8) is allocated on the heap, except for small integers: basically integers that can fit into a pointer. However, there's a catch: on 32-bit platforms, our integer has to fit into 31 bits because V8 uses 1 bit to distinguish between a pointer and an actual number. And on 64-bit platforms, our integer still has to fit into 32 bits because of [pointer compression](https://v8.dev/blog/pointer-compression). This is not a problem for us: 15 bits is still more than we need - the biggest map I've planned is like 200x200. One thing to note is that in JavaScript, there's no "integer" - all numbers are double-precision floats. So this is essentially a vendor-specific optimization. However, V8 is very commonly used and the game is shipped in Electron, which makes this optimization justifiable.

In the case of generating a 31-bit integer key for cache/memoization, it's more tricky. The majority of the game's data is in string - and it's preferable that way. After all, we tweak the data very often and strings are much easier to read. Instead, when the game starts, we go through all the data and generate an auto-incremental numeric key for each item. We also use enum flags in TypeScript instead of an object of boolean flags. An example is:

```typescript
// Before
function calculate(building: string, options: { flag1: boolean; flag2: boolean; flag3: boolean }) {}

// After
export enum Options {
    None = 0, Flag1 = 1 << 0, Flag2 = 1 << 1, Flag3 = 1 << 2, TotalBits = 3,
}

function calculate(building: string, options: Options) {
   const hash = (BuildingHash[building] << Options.TotalBits) | options;
}
```

## Adopting Map/Set (With Numeric Keys)

I have to admit that **I am not a JavaScript performance expert**: I do not write high-performance JavaScript professionally (I do write high-performance code but in other languages) and I do not really follow the latest bells and whistles in JavaScript world. I am vaguely aware of the existence of Map/Set in JavaScript but I've always used the plain old `object`. So I did a quick micro-benchmark of a slice of the common usage pattern in the game.

```javascript
new Bench
.add("access obj (number key)", () => {
   let sum = 0;
   for (let i = 0; i < 1000; i++) {
      sum += numObj[(i << 16) | i];
   }
})
.add("access map (number key)", () => {
   let sum = 0;
   for (let i = 0; i < 1000; i++) {
      sum += numMap.get((i << 16) | i);
   }
})
.add("access obj (string key)", () => {
   let sum = 0;
   for (let i = 0; i < 1000; i++) {
      sum += strObj[`${i},${i}`];
   }
})
.add("access map (string key)", () => {
   let sum = 0;
   for (let i = 0; i < 1000; i++) {
      sum += strMap.get(`${i},${i}`);
   }
});
```

The ranking of the result is not that surprising, but the difference is! I know numeric (small integer) keys will be faster, but didn't expect *that much faster*. Also `Map` with numeric keys is the winning combo here, which means the hard work of replacing strings with small integers provides us with some extra payoff!

```
┌───────────────────────────┬───────────┬────────────────────┬──────────┬─────────┐
│         Task Name         │  ops/sec  │ Average Time (ns)  │  Margin  │ Samples │
├───────────────────────────┼───────────┼────────────────────┼──────────┼─────────┤
│ 'access obj (number key)' │ '60,121'  │ 16632.928519960293 │ '±0.59%' │  6013   │
│ 'access map (number key)' │ '245,327' │ 4076.1832382745756 │ '±0.65%' │  24533  │
│ 'access obj (string key)' │ '10,316'  │ 96930.13813144476  │ '±4.19%' │  1032   │
│ 'access map (string key)' │ '17,543'  │ 57001.994138429654 │ '±1.53%' │  1755   │
└───────────────────────────┴───────────┴────────────────────┴──────────┴─────────┘
```
I've also done a benchmark of iterating maps vs objects:

```javascript
new Bench
.add("iterate obj (number key)", () => {
   let sum = 0;
   for (const key in numObj) {
      sum += numObj[key];
   }
})
.add("iterate map (number key)", () => {
   let sum = 0;
   for (const [_, value] of numMap) {
      sum += value;
   }
})
.add("forEach map (number key)", () => {
   let sum = 0;
   numMap.forEach((val) => {
      sum += val;
   });
})
.add("iterate obj (string key)", () => {
   let sum = 0;
   for (const key in strObj) {
      sum += strObj[key];
   }
})
.add("iterate map (string key)", () => {
   let sum = 0;
   for (const [_, value] of strMap) {
      sum += value;
   }
})
```
And here's the result:

```
┌────────────────────────────┬───────────┬────────────────────┬──────────┬─────────┐
│         Task Name          │  ops/sec  │ Average Time (ns)  │  Margin  │ Samples │
├────────────────────────────┼───────────┼────────────────────┼──────────┼─────────┤
│ 'iterate obj (number key)' │  '5,769'  │ 173319.23769086445 │ '±1.42%' │   577   │
│ 'iterate map (number key)' │ '178,355' │ 5606.781289537164  │ '±1.94%' │  17858  │
│ 'forEach map (number key)' │ '185,631' │ 5387.017655490776  │ '±0.55%' │  18564  │
│ 'iterate obj (string key)' │ '25,965'  │ 38512.431584109334 │ '±1.42%' │  2598   │
│ 'iterate map (string key)' │ '179,725' │ 5564.040455324439  │ '±2.05%' │  17973  │
└────────────────────────────┴───────────┴────────────────────┴──────────┴─────────┘
```
Map with numeric keys is again the winner (although the fact that iterating an object with numeric keys is that much slower is a bit puzzling)

## A Word Of Caution

Getting rid of strings and replacing objects with `Map/Set` together shaves another ~3s from the overall time. At this point, we've reduced the overall time from 21s to 4s. However, these two optimizations require a lot of work - it took days and I had to deal with several nasty bugs. Also, I find doing micro-optimization in JavaScript much harder compared to languages like C/C++:

1. Micro-benchmarking can be unreliable: JIT needs warm-up. The garbage collector cannot be turned off.
2. Optimization is done against a specific implementation in V8, which can change. And figuring out the internals of V8 is not trivial - reading the source code is not easy. And the several layers of optimizing JITs only make it harder.
3. Most useful micro-optimizations are usually data locality-related (e.g. making data smaller so more can fit into a cache line or making sure data that are accessed together are located together), which unfortunately is hard to achieve in JavaScript.

Apart from the "success stories", I have also tried a bunch of micro-optimizations that didn't move the needle.

Changing from a numeric array to a typed array (`Uint32Array`) does not help:

```javascript
new Bench()
.add("access array", () => {
   let sum = 0;
   for (let i = 0; i < 1000; i++) {
      sum += numArray[i];
   }
})
.add("access uint32array", () => {
   let sum = 0;
   for (let i = 0; i < 1000; i++) {
      sum += uint32Array[i];
   }
});
```
```
┌──────────────────────┬─────────────┬───────────────────┬──────────┬─────────┐
│      Task Name       │   ops/sec   │ Average Time (ns) │  Margin  │ Samples │
├──────────────────────┼─────────────┼───────────────────┼──────────┼─────────┤
│    'access array'    │ '1,824,996' │ 547.9462917536906 │ '±0.35%' │ 182500  │
│ 'access uint32array' │ '1,828,721' │ 546.8302087845923 │ '±0.17%' │ 182873  │
└──────────────────────┴─────────────┴───────────────────┴──────────┴─────────┘
```

Replacing a branch with branchless code doesn't help that much:
```javascript
// With branch
const result = x > y ? (x << 16) | y : (y << 16) | x;
// Without branch
const result = (Math.max(x, y) << 16) | Math.min(x, y);
```
```
┌──────────────┬──────────┬────────────────────┬──────────┬─────────┐
│  Task Name   │ ops/sec  │ Average Time (ns)  │  Margin  │ Samples │
├──────────────┼──────────┼────────────────────┼──────────┼─────────┤
│ 'branchless' │ '53,060' │ 18846.260416366207 │ '±1.45%' │  5307   │
│   'branch'   │ '40,458' │ 24716.880848309927 │ '±1.20%' │  4046   │
└──────────────┴──────────┴────────────────────┴──────────┴─────────┘
```

## Premature Optimization is the Root of All Evil?

Donald Knuth is often quoted out of context. People use this quote to justify that:

1. Performance doesn't matter, computers are fast enough
2. Write clean code first and performance can always be optimized later

> Programmers waste enormous amounts of time thinking about, or worrying about, the speed of noncritical parts of their programs, and these attempts at efficiency actually have a strong negative impact when debugging and maintenance are considered. We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil. Yet we should not pass up our opportunities in that critical 3%.

Among all the optimizations, I feel the first two (getting rid of excess allocation in one hot function and memoization) are good examples of "optimizing later". The latter two, however, would be much easier if they were done upfront. Also, the latter two are [uniformly slow code](https://wiki.c2.com/?UniformlySlowCode) that do not really show up in the profiler: things like how a tile is represented are so fundamental and used everywhere. Those small inefficiencies are spread throughout the code base, making it even harder to *optimize later*.

Context matters a lot when deciding when to apply optimizations. In game programming, it's usually easy to identify hot code paths even when writing the initial code - and adopting some baseline performance practice usually proves fruitful. This also brings up the point of language and ecosystem: in general, there are more performance-minded (and allocation-minded) libraries in C++ than in JavaScript. Oftentimes, a library in C++ can be used without modifications but in JavaScript, most libraries are not written for performance-critical software. Ironically, it's more common to vendor a C++ library due to the lack of a *de facto* package manager.

That's all I have for now: thanks for reading. I want to reiterate that I am not an expert in JavaScript performance. So If you've spotted an error or some obvious low-hanging fruit that I've missed, please let me know. And If CivIdle sounds like a game that you are interested in, head over to [Steam](https://store.steampowered.com/app/2181940/CivIdle/) and check it out.