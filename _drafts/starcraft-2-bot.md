---
layout: post
title: Making A StarCraft II Bot
---

I participated in a [StartCraft II bot competition](https://artificial-overmind.reaktor.com/) in Finland. This is my first attempt at a StarCraft II bot, although I have written a [Dota 2 Bot](https://github.com/insraq/dota2bots) before. And compared to Dota 2, writing a StarCraft II bot is quite pleasant in terms of development setup.

You need to download the game client, which is already free, and choose a library. [python-sc2](https://github.com/Dentosal/python-sc2) is an excellent starting point. You can simply run the Python script and the bot match will automatically start - as simple as that. In comparison, I need to document all the quirks and tricks used to set up a [Dota 2 bot](https://ruoyusun.com/2017/01/08/dota2-ai-quickstart.html).

*python-sc2* actually does quite a bit of heavy-lifting for writing a high-level bot: expansion locations are calculated for you, a fairly comprehensive unit selection API is provided. I have put the [source code of my bot on Github](https://github.com/insraq/BaseTrade). It's not pretty: everything in one `main.py` file but it should be relatively straightforward to understand.

