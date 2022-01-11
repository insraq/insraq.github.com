---
layout: post
title: A Poor Indie's Journey To Developing And Running A Multiplayer Game
---

[Industry Idle](https://industryidle.com/) is a resource management game inspired by lots of great games in this genre: Factorio, Satisfactory, Offworld Trading Company, you name it. There are two nice additions: idle game prestige mechanism and player-driven trading system. The former earns the "idle" in the game's title, although the core gameplay is not "idle" at all. The latter opens a whole can of worms - especially for a one-man indie, who works on the game as a hobby.

I've worked on different game productions - big and small. [Netcode](https://ruoyusun.com/2019/03/28/game-networking-1.html) is usually my expertise in the team. However, it's quite different when you have a big team supporting you and a decent budget to spend on servers - I have neither for Industry Idle.

Cost (development cost + running cost) and complexity are two main reasons why most indies shy away from multiplayer game servers, which is a very sensible thing to do. Here I want to provide my experience while developing and running Industry Idle.

## Start With No Multiplayer

Industry Idle started with single-player only. Even though a global player-driven market has always been part of the vision (the game starts with the idea what if you can trade resources with other players in Factorio), I was not sure whether the game would be successful enough (i.e. have enough players) to have a player-driven market.

I've said before in an [article](https://ruoyusun.com/2020/04/01/multiplayer-beginners-guide.html) that "..._if your core gameplay involves multiplayer (i.e. your game will not be shippable and complete without multiplayer), you should start with multiplayer code._" I think this still holds true here. I decided that I should ship a fun game first without multiplayer first. I know this is possible because lots of games in this genre are fun in single-player mode.

When I look back now, this is the correct decision as it allowed me to ship the game in 3 months. At that time, it was Covid lockdown so there was nothing else to do - that helped me stay laser-focused as well.

## First Multiplayer Prototype

The game attracted a small but fervent fan base after the initial beta release. After most of the nasty core gameplay bugs were gone, I feel it was time to add a multiplayer trading system.

I knew dedicated server code would be inevitable eventually but I decided to postpone it as long as possible because of the complexity it brought to the development (and release) process. So the first multiplayer prototype _had no server code at all_. The centralized data was stored in [CouchDB](https://couchdb.apache.org/) and all logic was implemented on the client-side.

I chose CouchDB because its HTTP API made development easy. Not having server code was obviously a big trade-off here. After all, the client was reading and writing directly to the production database with minimum authentication! Also, HTTP is definitely not well suited for a real-time trading server - to "simulate" real-time updates, I had to rely on short polling, which is wasteful. I would never do this in commercial game production. But it allowed me to ship the first prototype of the feature in a week. I also told the game community that consider this mechanism as experimental and it might be changed or even removed later. All I want to know is whether this would make the game more fun or not.

## Server For Multiplayer

I could get away with no server code, but I couldn't get away with servers - I needed a machine to run CouchDB. Spinning up a €10 per month virtual private server would be the sensible thing to do here and I didn't do that.

Because this was a hobby project and I wanted to keep things more fun and give myself a challenge. I decided to run the server from my comfortable home!

![](https://de.catbox.moe/io5o1u.jpg)

I had this old laptop running in my living room TV cabinet as the "entertainment system" (streaming Netflix). Most of the time it was idling so I figured I could give it more load. There were two problems left to solve:

1. The laptop was running Windows 10 Home which works okay for entertainment but was not good for running a server

2. My [home broadband](https://www.dna.fi/) has a dynamic-ish IP. Once in a while when the router restarts, I get a new IP. Although that doesn't happen too often, I cannot risk it. But the good news is that there's no _Carrier-grade NAT_ so I do get a public IP.

I briefly considered running a Linux VM but the processing power would be a bottleneck as the laptop came with an Intel U series CPU. In the end, I decided to bite the bullet and run the server on Windows. Fortunately, CouchDB has decent support on Windows - there's prebuilt binary, and getting it up and running only requires .Net Framework 3.5.

My router (which sits on top of the laptop) doesn't have good Dynamic DNS support so I had to write a custom script that updates the DNS record (I use Cloudflare DNS). The script runs every hour using "Windows Task Scheduler", which took quite a bit of googling for someone who never used this before.

In the end, everything worked out - I got an essentially "free" server (not really free but I am not paying the extra cost). The laptop fan sometimes got really loud so I went to laptop BIOS to disable Turbo Boost. Obviously, that cost some performance it is probably still better than those €10 VPS.

## Introducing Server Code

The player trading system is very well received by the players - it adds another level of strategy to the gameplay and becomes a major feature and unique selling point of the game.

However, my prototype implementation now ran into issues:

-   **Cheating**: with little authentication, cheaters could seriously disrupt the experience for everyone. I had to keep going to the database and cleaning up the mess.

-   **Performance**: as mentioned before, HTTP REST API is not great for a real-time trading system. Plus the hardware is not powerful enough.

-   **Gameplay**: there are several additions to the gameplay that would require server code. Plus polling CouchDB to simulate the "real-time" market is not good enough anymore.

The game's client code is written in TypeScript so it makes sense to [write the server in the same language](https://ruoyusun.com/2019/09/30/game-networking-6.html#server-programming-language). I briefly considered trying out a new language (mainly Rust) for fun but in the end, I went with a more sensible choice: Node.JS with TypeScript. The server was a separate project and the shared TypeScript code was symlinked.

The game's client code was developed on my Macbook Pro. Developing the server on a Mac is a sensible choice because most servers are deployed on Linux - and macOS is close enough.

However, my server is running Windows in my living room! Running Docker wasn't really viable either because my aging Macbook Pro was struggling with running the game's client alone. Adding a server would probably be okay; adding a server in a VM would definitely kill it. Plus my server wasn't powerful enough to run VM either.

So I decided to keep the dependency to a minimum - the server should be self-contained - that meant no dependency on Redis/Memcached. All data was stored in memory and dumped to a file on a regular schedule and during the server shutdown routine. When the server started up, it loaded the dump file into memory.

This design obviously has pros and cons. One of the biggest cons is that it's almost impossible to scale horizontally - but it's okay in this case since I cannot afford to do that anyway.

I've chosen WebSocket as the communication protocol. Mostly because I need to support playing from the browser (the game's iOS, Android, Linux, Mac, and Windows ports all run inside WebView/Electron). But even if I don't have to support browser, I would still go with [TCP instead of UDP](https://ruoyusun.com/2019/09/30/game-networking-6.html#tcp-vs-udp):

-   The player trade is fast-paced, but not _that_ fast-paced. After HTTP short polling was deemed acceptable before.

-   The reliability layer of TCP means one less problem to worry about. In a fast-paced FPS, for example, it can be problematic because the head of line blocking can cause significant issues. For a player trade server, it is less so.

-   Most ISPs and carriers are friendlier to WebSocket/TCP. According to my past experience, UDP traffic gets more hostile treatment.

I implemented a WebSocket server instead of using a higher abstraction library like _socket.io_ because I feel the extra abstraction is not needed and the overhead for each packet is a bit wasteful. But if I am honest, I could use _socket.io_ and it probably will be fine.

## Evolving The Server Code

The first version of the server is "dumb" - whenever there's a change in player trade, the server sends out all the active trades to every client. It was done like that because in the earlier HTTP polling version, the client queries all active trades and then figures out the change, notifies the player, and shows in the UI.

Keeping this allows _minimum change_ on the client-side - instead of polling, the server is pushing the same data. This is also easy to implement on the server-side.

Here the important lesson is that to make complexity manageable, **do one thing at a time**. Sever migration is complex but manageable. Server optimization is also complex. Doing server migration and optimization all in one go would be hard to manage (although it is very tempting). Divide a big problem into smaller ones and conquer them one by one!

After the migration was completed, optimizations were delivered in subsequent game updates - the server would only send delta to clients that needed to be notified.

Adding server code does introduce more complexity in the development process. One annoying issue is to make sure the client and the server are "compatible" if I make changes. The game supports multiple platforms and all of them share the same server. Coordinating iOS, Android, Web, and Steam releases to be exactly the same time is almost impossible. So instead I have to make the server capable of working with a different version of the client. There are lots of ugly `if (client.version === "v1")` in the server code but there isn't a better alternative.

After WebSocket migration, I initially went all in - all the communication between the serve and client went through WebSocket. Later I realized that for some features, HTTP's Request and Response model is a better fit. Oftentimes the client clicks a button, the server does some processing and returns the result and the client displays it. I added a layer on top of WebSocket to support this pattern - it worked but it made development harder. Then I realized I could just provide HTTP REST APIs on the server. I was running a simple HTTP server anyway for the initial WebSocket upgrade request, why not use it for other requests as well? On the server, I could use Express.js and its mature ecosystem. On the client, I could stick to `fetch` instead of my homegrown abstraction.

## Moving The Server Out of My Living Room

Finally, I've come to the conclusion that running a game server in my living room is just "too much". Here were the problems:

-   Network bandwidth became a bottleneck. My home broadband is 100Mbit/s. It would be more than enough for running the server alone. However, whenever I watch Netflix or download a system update, the bandwidth is maxed out, which means the game server's packets are inevitably affected.

-   Hardware was overloaded. The laptop has a 6 core CPU (U series with Turbo Boost turned off) and 16GB RAM. However, there are simply too many services running that makes it slow during peak load:

    -   Decoding video and output to my 4K TV

    -   Doing CouchDB indexing / compacting (yes, the CouchDB is still running even though all "hot" data is stored in RAM. Cold data like audit logs and paper trails are stored in CouchDB)

    -   Building Windows/Android version of the game (my main laptop is a Macbook Pro, which builds the Mac, Linux, and iOS version of the game. Windows/Linux builds are done on this Windows laptop)

-   Running a Windows server felt weird. It was probably just me so used to dealing with a Linux server. I didn't realize that [Windows 10 supported OpenSSH](https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse). So for me deploying a new server version was like connecting to the server using Remote Desktop or better yet, walking to my living room and turning on the TV. I did "share" the server log file directory via SMB so that I could view the log on my Mac easily.

The cost of the server is definitely a concern since the game makes very little money: the base game is free and there's an optional DLC that costs €3.99. I don't plan to make a profit from this game so my goal is to make the game self-sustainable. _The game's income should cover the monthly running cost_. After calculating the game's proceed after Steam's cut, forex fee, bank's fee, and tax, accounting for the diminishing income as time goes by and leaving a little headroom, I have reached the conclusion that the server cost should be around €15 per month.

With this budget in mind, I went on searching for cheap server options. The option with the best price/performance ratio is ... _Oracle Cloud_. I know what you must be thinking: Oracle, no way! I agree - I was never a fan of Oracle but I simply couldn't say no to their generous free quota and aggressively low pricing. There was one catch though - it was ARM-based.

I briefly tested ARM-based server several years ago while working on commercial game production. The goal was to optimize the server cost. At that time I tried running the server which was written in .Net Core (before .Net Core would evolve to become .Net) on AWS ARM instances and benchmarked it against x86 instances. The result was less than impressive - for the same CPU/RAM spec, the ARM could only handle about 50% of what the x86 counterpart could. I didn't have time to look into the cause - it could be hardware, .Net Core ARM support, or the codebase itself but the conclusion was that it was more economical to stick to x86.

I should also mention that around the same time, I switched my main development machine from Macbook Pro (Intel) to a custom-built PC. I finally got tired of the fan noise and throttled CPU performance. The new PC is equipped Ryzen 5700G. Since I couldn't find a graphics card at a reasonable price, I decided to do development on the integrated graphics, which is less than ideal but fortunately, the game isn't GPU intensive anyway. That means for a brief period of time, I was doing server development on Windows and deploying it on Windows as well - but not for long!

## Migrating A Stateful Server

I decided to get stuff up and running on an Oracle ARM instance first. Installing Node.JS is relatively straightforward - there's an ARM build for the OS package manager already. Within 10 minutes, I can connect to the game server from my local game client.

Getting CouchDB running was far more difficult. First I realized that CouchDB only provided prebuilt packages for Debian. Oracle ARM instances officially only supported Ubuntu / Oracle Linux and I was running Ubuntu. I tried using Debian packages on Ubuntu but was of no avail. So compiling from source code seemed the only way. Getting all the dependencies ready was already painful and I had to patch the CouchDB [configure](https://github.com/apache/couchdb/blob/main/configure#L214) to allow SpiderMonkey 68, which came with Ubuntu 20.04.

After getting everything up and running (including SSL certificates), now it was time for the actual migration. Migrating a stateless HTTP server is easy. Assuming the actual database remains the same, I can have both instances running, switch the DNS from the old server to the new one and traffic will gradually flow to the new server because DNS change takes anywhere between a few minutes to days. Because my game server stores everything in RAM, doing this would result in some players connecting to the old server and some to the new one, which will cause data corruption - I have to make sure all traffic is immediately sent to the new server when I flip the switch. This requires the following careful steps:

1. Set up a reverse proxy (e.g. Nginx) on the old server that forwards all connections to the new server, but doesn't run it yet!

2. Shut down the old server, all clients will disconnect, which is expected. The client has reconnected mechanism because, during normal deployment, I need to restart the server, which would require clients to reconnect.

3. Copy the memory dump file (i.e. data) across to the new server and spin up the new server.

4. Turn on the reverse proxy on the old server. I need to do steps 2-4 as quickly as possible because during that time the server is down.

5. Change DNS to the new server. The clients with updated DNS will connect to the new server directly. The clients with outdated DNS will connect to the new server via the reverse proxy on the old server.

6. After a few days, when the old server no longer has any traffic, shut down the reverse proxy.

## Running Linux Server on WSL

Now that the actual game server is running somewhere in the data center, I feel that my living room laptop is underutilized. I've decided to run some development tools and services on that machine. Normally in a commercial setting, you would have application performance monitoring, error reporting, infrastructure, and uptime monitoring services to make your life easier. Obviously, as a poor indie, I don't have that luxury. But now since I have some extra processing power to spare, it's time to add some quality of life tools.

I feel the best value out of those tools is error reporting - which allows me to proactively fix runtime errors and exceptions, instead of relying on people reporting bugs and crashes on [Discord](https://discord.com/invite/m5JWZtEKMZ). These SaaS products are not cheap but fortunately one of the best error reporting services [Sentry](https://sentry.io/) provides a self-hosted option. The default installation doesn't work on Windows though (again, who runs a Windows server!). I briefly read through the scripts and I could probably get it up and running on Windows natively because it is mostly running in Docker. But there are a whole bunch of dependencies and steps involved so I feel it's easier to run it in WSL (Windows Subsystem for Linux).

I haven't used WSL before (I've been a long-time Mac user) but from what I understand, WSL2 takes a different approach compared to WSL1 - it's essentially a lightweight VM, which means compatibility with Linux shouldn't be an issue. However, running a public server from WSL2 isn't as straightforward. A request needs to go through all these layers.

```
Docker -> WSL2 VM -> Windows -> Router
```

I've followed [this instruction](https://www.williamjbowman.com/blog/2020/04/25/running-a-public-server-from-wsl-2/) to expose the service on the public Internet (yes, you need to do all these steps, don't skip any of them!)

With WSL2, I've also added a cron job (say goodbye to Task Scheduler) that does production data backup as well.

## Lessons Learned

Servers add complexity - this is very true, even for me who has quite a lot of experience working with servers. Here are some lessons learned:

-   Keep things simple. Scale vertically until you cannot. You'd be surprised how far that can get you.
-   Divide and conquer. Instead of trying to achieve everything in one go, do it in steps.
-   Ship something good enough. If you ship something "perfect", you are shipping too late. And your "perfect" solution will quickly get outdated as your game evolves.
-   Stick to mainstream solutions unless you are prepared to go knee-deep in its internals.
-   And most importantly, **do not host your server in your living room**, no matter how cool that sounds.
