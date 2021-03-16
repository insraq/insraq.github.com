---
layout: post
title: Kotlin is A Good Glue (as Opposed to Scala)
---

Almost 8 years ago, after working with Scala in a large Java codebase, I wrote [Scala is Not Java Glue](https://ruoyusun.com/2013/11/10/scala-is-not-java-glue.html). Scala was the "hype language" at that time and that article inevitably triggered some discussions. To be fair, Scala is not designed to be a glue language for Java and using it as one will of course backfire.

Recently, I needed to do an Android port for my game [Industry Idle](https://industryidle.com/). The platform specific code is very thin: mostly to bridge some native APIs and 3rd party SDKs to my game code. I initially planned to use Java as I prefer my bridge code not to have too many layers, which might make debugging and error tracing more difficult. But I decided to give Kotlin a try — on the assumption that I can quickly switch back to Java once things start to fall apart.

And I am happy I never had to. Kotlin turns out to be a fantastic language to work with, especially on Android. And working with existing Java libraries and code is quite smooth.

## Seamless Interoperability

Calling Java code in Kotlin feels natural. Most things just work as you expect, without needing extra conversions or bridging. I initially thought *Null Safety* will make calling Java code quite bloated but it turns out Kotlin's *platform type* approach works most of the time - and I rarely need to worry about it.

I have done several language migrations recently. And to rate the level of disruptions, I'd say: JavaScript/TypeScript < Java/Kotlin < ObjC/Swift. Obviously migrating from Java to Kotlin isn't as easy as "change the file name extension and things will work". But it isn't very far off. The "Convert Java to Kotlin" provided by Jetbrains IDE works really well. And when I paste some Java example code from 3rd party SDK documentation, it also converts to Kotlin automatically. These neat little features really make a big difference in terms of making the transition as smooth as possible

## Flat Learning Curve

For a Java programmer (or in my case, someone who did some Java like 10 years ago), picking up the language is relatively easy (at least compared to Scala). When I started to work on my game's Android port, I didn't even bother reading the Kotlin documentation. Instead, I dive straight into the code. And if I encounter some syntax that I don't quite understand, I pull out the Kotlin documentation. This approach generally works quite well for me.

Although Kotlin can be made more "similar" to Java (I am looking at you C#) but I feel that most of the changes are well justified — they are easy to learn and provide very tangible benefits. The language itself is quite concise and expressive.

## Good IDE/Tooling/Ecosystem

The fact that Kotlin comes from Jetbrains should pretty much guarantees a good IDE support out of the box. The compiler is relatively fast, at least I never feel a big difference compared to Java.

Ever since Google adopted Kotlin as one of the official languages of Android, the ecosystem has grown quite a bit. Using Java library in Kotlin is already easy. Now many libraries even have a Kotlin version, which unlocks more possibilities.

To be fair, Java has improved a lot recently, with [Java 16](http://openjdk.java.net/projects/jdk/16/) coming out very soon. However, I still feel that learning and using Kotlin is definitely worth the effort, even for a project that has a deadline to deliver.