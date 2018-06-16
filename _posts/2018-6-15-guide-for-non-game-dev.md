---
layout: post
title: What I've Learnt After Making My First Hobby Game
---

## Why

A lot of people (myself included) get into software development because of computer games — as a kid, I was amazed at computer games and always curious about how it was made. However, after finally getting into the industry, most people (myself included) end up not doing game development.

Extra Credit has an interesting video on [Non-Professional Game Dev](https://www.youtube.com/watch?v=m4p7T9O_tqg), which has a good description of hobby games — they are made mainly for the people who's made them. For me, I make hobby games:

- **To make my childhood dream come true**
- **To have fun:** making stuff is fun and gives a sense of accomplishment
- **To get to know game development:** game development is quite different from software development. Although I have other hobby projects that are more similar to my day time programming, it feels really nice to take a break and try something different
- **To learn something new other than writing code:** I've learnt to write stories, make pixel art and make music and sound effect.

## Aim Small

When you make your first game, aim small, I mean *really small*. Otherwise you won't be able to finish your game. Forget about AAA games — they costs millions of dollars and hundreds of professionals to make. Even indie games nowadays costs tens of thousands at least. And ancient games, like the first Zelda, has too much content as a first game. Here are some examples of well-scoped first games:

- **Puzzle:** it's a great genre to be creative and experiment different ideas. As long as you can make the core mechanism works for one level, it's easy to build upon it and expand your game.
- **Match-3:** Even though the core mechanism is pretty much set, you still have plenty of room to extend. It's a safe genre to experiment without worrying about the core mechanism.
- **Arcade:** A great way to design your first game is to start with your favorite arcade game and change only *one* mechanism.
- **Platformer or Runner:** Try to limit the scope, either by having fewer mechanisms or small levels.

As to whether you should make your first game in 2D or 3D, the answer is whatever you feel most comfortable with. 3D might be a bit more complicated. But if you are really good at making 3D models, that will work out as well.

Also, multiplayer dramatically increases the complexity, so try to avoid it. There are those ["IO"](http://agar.io/) games that very simple mechanism and graphics but they might still be a bit overkill for your first game.

## Make a Plan

I know in software development, waterfall approach is not widely used. But when developing your first game, it's good to make a "master plan". The goal of the plan though, is not to follow it strictly, but rather to have a good idea of the scope and progress. When making the plan, use your best estimate (or guesstimate). Your planned schedule should be within 1-2 months, otherwise you need to cut your scope. You will have more difficulty finishing the game if the plan is longer than 2 months. And also, you will most likely spend double the planned time.

Having a plan also helps with "finishing" the game. As happens to most hobby projects, the scope can keep expanding. You will always want to add more content, a new mechanism and optimize and refactor the code. And having a plan will help your game reach "feature freeze" so that you can release it.

## Choose Your Tools

There are tons of tools for making games. So *don't make your own engine for your first game unless making your engine is the main goal*. And making your own engine can easily double or triple the development time thus increasing the risk of not finishing your game.

When choosing a game engine, the programming language most likely will **not** matter. Because coding (or commonly known as *scripting*) in the game engine is more about engine APIs (like transform, camera, physics, etc) and game data. Programming language seldom posts a challenge. If you know golang, picking a golang based game engine is less likely to save you much time. Instead, pick a mature game engine and use whatever scripting language the engine supports. That being said, I find it a bit easier to work with a static type language as the typing itself provides a good API documentation so that I don't have to keep going back to the API reference webpage.

**Try to go with an engine that has an editor.** I know coming from software development, you might be skeptical about drag and drop and WYSIWYG editors. But game is more about visuals and having an editor is a great time saver. Not only does it speed up the feedback loop, it also makes your code much cleaner (and shorter).

**Prefer a stable and mature game engine with an active community.** The last thing you need while making your first game is that the game engine breaks its API or you encounter an error and cannot find help. I know there are lots of new game engines being developed that looks exciting or promising but remember, your goal is to finish the game, not to participate in the engine development.

Lastly, **check out a demo project from the engine you've chosen.** It's the fastest way to get to know the API and you can learn a lot. You might even base your first game on the demo project.

## Track Every Week

Procrastination is your biggest enemy. For a project with deadline, procrastination means crunch time before deadline. For your hobby game, which does not have a deadline, procrastination pretty much kills it. here are some tips to avoid it:

- **Set up a task tracking system**. Once you've made a master plan, break it down and get it into whatever task tracking system (e.g. todo list or Trello board) you use.
- **Touch the code every week**. Even some weeks that you are super busy, at least try to touch the code: fixing typos and small bugs. Once you've made it a habit, it's easier to keep it on.
- **Show the progress every week.** Having a group of "gamedev" friends and check progress with each other every week not only helps with procrastination, but also provides a good way to get some early feedback. Keeping a **devlog** or participating in **#screenshotsaturday** also helps. And you can get to know more game developers and even get some prelaunch marketing for your game.

## The Last Mile

The most difficult time for hobby game development is towards the "end". Because at first, there are lots of exciting stuff: new ideas, new game engine, new experiments, etc. But as time goes by, you will run out of "interesting" tasks and those boring and tedious tasks you've been avoiding become unavoidable — that's the point where most people give up the game and never ship it.

So when you are reaching your goals, it's not time to relax, rather, you need to be more disciplined. To push yourself finishing all those boring tasks, try to get the early access builds to a wider audience. Sometimes cruel feedbacks are the best stimulus to finish the last mile.

It also helps to have a formal "release date" and put it out there, make a trailer or poster with a release date, have a countdown, plan a release party. These might sound symbolic, but you will be much more likely to finish the game.

For my first games, I choose to make several Facebook Instant Games because it's easier to share and play with my friends, you can check them out: [Pinball Breakout](https://www.facebook.com/instantgames/198493907436001/), [Box Flip](https://www.facebook.com/instantgames/1386176524819862/), [Rocket Run](https://www.facebook.com/instantgames/179283319572478/)