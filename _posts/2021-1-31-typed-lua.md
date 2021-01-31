---
layout: post
title: Lua Game Development And Type Safety
---

I am working on my next game project: [**Idle Space Factory**](https://twitter.com/insraq/status/1355625568545468417). It's a game combining resource management, factory building and trading. I have been using TypeScript for all my games for a while and I couldn't be more happier. The gradual and [structural typing](https://www.typescriptlang.org/docs/handbook/type-compatibility.html) system, plus the powerful [type inference](https://www.typescriptlang.org/docs/handbook/type-inference.html) make TypeScript a great fit for game development. In the early prototyping phase, I don't have to bother with the type declarations, since they will change anyway. And when I've "found the fun" and start production, I can add the types and enjoy the type safety.

However, TypeScript/JavaScript is not very commonly used as the scripting language for games. For this purpose, Lua is more widely used. Lua is like JavaScript's weird cousin, built especially for embedding into C/C++ codebase. A big Lua codebase is very hard to maintain. Also when I work on game mods, I have to constantly look at the documentation (if there's any) to explore the API surface. It would be much nicer if it is built into the editor's intellisense. I've been looking at and trying out several "Typed Lua" transpilers for different games.

## Haxe

[Haxe](https://haxe.org/manual/target-lua-getting-started.html) is probably one of the first transpilers that provide a Lua target. Since Haxe itself has been out for a while and are widely used in game development, the IDE and tooling support is relatively good. Haxe's language feature and standard library is quite big, therefore the Lua target is a much smaller subset. So using it in a embedded game might not be as straightforward. The transpiled source is hard to read and oftentimes embedded Lua requires exposing certain functions/variables from script as the entrance, this adds to the difficulty.

## Teal

[Teal](https://github.com/teal-language/tl) comes from the author of LuaRocks. It's relatively new and use a minimalist approach when it comes adding static typing, which makes is very suitable for using in embedded Lua environment. The language itself has some influence from TypeScript when it comes to introducing a type system for a dynamic language. Since it is quite new, the features are still changing and the IDE/tooling support is still catching up. But even using it as is, it already provides benefits to existing Lua codebase.

## TypeScript

I have discovered [TypeScriptToLua ](https://typescripttolua.github.io/) while writing a Dota 2 Bot a while ago. At that time it was still quite new and the transpiler has matured quite a bit since then. Building on top of TypeScript brings the benefit of top notch IDE/tooling and TypeScript's superior type checking system. There are of course [caveats](https://typescripttolua.github.io/docs/caveats) like Haxe as well. But arguably, JavaScript is definitely closer to Lua compared to Haxe. The transpiled source is still readable and it makes working with embedded Lua much easier. And people have been providing type [declarations for popular embedded environment](https://typescripttolua.github.io/docs/getting-started#declarations) already. However, it is important to keep in mind that not all TypeScript features are supported because of language difference.

It's been a while since I work with Lua and I am definitely happier working with TypeScript nowadays. If you are currently working with Lua and want more type safety, I recommended checking these out.