---
title: Creating a Discord Bot for Music Poster Generation
date: 2025-05-29 08:00:00 +0530
categories:
  - Blog
  - Programming
tags:
  - python
  - discord
  - bot
  - beatprints
  - railway
description: How to create a host a discord bot using python and railway
---
>Make sure the python script is running and the bot is up before trying out the commands
{: .prompt-warning }
## Installing the dependencies
For creating the discord bot and for its functionalities, we are going to use the following libraries :
- discord.py - Wrapper around the discord API
- BeatPrints - For poster generation
- python-dotenv - For fetching env variables without exposing personal token
- You can install these with :
```python
pip install discord.py BeatPrints python-dotenv
```
- Make sure to create a venv with `python3 -m venv .venv` and to activate it before installing the dependencies.
## Creating a bot in Discord Developers Portal
- For creating a bot, first we need to create a bot in the discord developers portal and obtain the `BOT TOKEN`.
- For this navigate to [discord developers portal](https://discord.com/developers/applications) and create a new application.
- You can choose the name and provide an avatar image if you would like
- In the BOT section, you can find the bot token. Copy the bot token and save it to a `.env` file in the same folder as your python script.
- Also provide the bot with necessary intents (privileged gateway intents ). I usually toggle all of them on.
## Inviting the bot to a server
- Create a discord server if you don't already have one to test the bot.
- Now in the installation section in discord developers portal, you can find the install link.
- Before using the link, make sure you have selected the bot option in Guild Install menu. By default only the application.commands will be selected.
- Also select the necessary permissions below it. Let's give the bot admin permission so that it won't have any restrictions.
- Now you can use the Install Link to invite the bot to any server you have the permission to add a bot to.
## Creating a basic program
- Let's try running a basic program to test the bot :

```python
import discord
from discord.ext import commands
from dotenv import load_dotenv
import os

load_dotenv()
TOKEN = os.getenv("DISCORD_TOKEN")

intents = discord.Intents.default()
bot = commands.Bot(command_prefix="!", intents=intents)

@bot.event
async def on_ready():
    print(f"Logged in as {bot.user} (ID: {bot.user.id})")

@bot.command()
async def ping(ctx):
    await ctx.send("pong")

bot.run(TOKEN)
```
- Make sure in the `.env` file, you have pasted the TOKEN in the following format
```text
DISCORD_TOKEN=163jhdgl830XXXXXXXXXX
```
- Now in discord try testing the bot with `!ping` command. The bot should respond with pong.
![ping command](/assets/img/blog/programming/python/ping.png)
- If the bot is not responding verify the token and make sure the bot has proper intents and permissions.
- If the bot does reply, It means the token is correct and now we can add further functionalities to the bot.
## Using the bot for poster generation
- Let's modify the bot to take in a song name as input and generate a poster for it.
- For this we are going to use [BeatPrints](https://beatprints.readthedocs.io/en/latest/)
- Here is an example poster :

![example poster](/assets/img/blog/programming/python/beatprints-example.png){: w="300" }

- Follow the instructions given in BeatPrints wiki to get the Spotify client id and client secret [BeatPrints Github Repo](https://github.com/TrueMyst/BeatPrints) 
- After getting client id and client secret, paste it into the `.env` as we did earlier with the token with names `SPOTIFY_CLIENT_ID` and `SPOTIFY_CLIENT_SECRET`
### Code
- Instead of writing the whole logic into a single `main.py` file, we are going to separate these into differents cogs.
- Each cogs can have different functionality.
- For example, here is a folder structure with different cogs.

```bash
├─cogs
│ ├── devinfo.py
│ ├── help.py
│ ├── moderation.py
│ └── poster.py
├── flake.lock
├── flake.nix
├── main.py
└── requirements.txt
```

- Here there is a cogs folder with different cogs inside it. Each of which has a seperate functionality
- This makes it easier to debug and modularize code.
- So let's create a `cogs` folders and inside it a `poster.py` file. The cogs folder should be in the same level as the main.py file.

main.py

```python
import os
import discord
from discord.ext import commands
from dotenv import load_dotenv

load_dotenv()

TOKEN = os.getenv("DISCORD_TOKEN")

intents = discord.Intents.default()
intents.messages = True
intents.message_content = True

bot = commands.Bot(command_prefix="!", intents=intents, help_command=None)

@bot.event
async def on_ready():
    print(f"Logged in as {bot.user}")

# Load cogs
async def load_cogs():
	# Make sure the filename matches the cogs.<extension>
    await bot.load_extension("cogs.poster")

async def main():
    async with bot:
        await load_cogs()
        await bot.start(TOKEN)


@bot.command(name="ping")
async def ping(ctx):
    await ctx.send("Pong!")


import asyncio

asyncio.run(main())
```

poster.py

```python
import glob
import os
import shlex

import discord
from BeatPrints import lyrics, poster, spotify
from discord.ext import commands
from dotenv import load_dotenv

load_dotenv()

CLIENT_ID = os.getenv("SPOTIFY_CLIENT_ID")
CLIENT_SECRET = os.getenv("SPOTIFY_CLIENT_SECRET")


class BeatPrints(commands.Cog):
    def __init__(self, bot):
        self.bot = bot
        self.ly = lyrics.Lyrics()
        self.ps = poster.Poster("./")
        self.sp = spotify.Spotify(CLIENT_ID, CLIENT_SECRET)

    @commands.command(name="poster")
    async def generate_poster(self, ctx, *, args: str):
        # Send initial loading message
        loading_message = await ctx.send("🎨 Generating poster... Please wait.")

        try:
            # Parse arguments with shlex to support quoted song names
            tokens = shlex.split(args)
            line_range = "1-4"
            track_name_parts = []

            for token in tokens:
                if token.startswith("--lines="):
                    line_range = token.split("=", 1)[1]
                else:
                    track_name_parts.append(token)

            track_name = " ".join(track_name_parts)
            if not track_name:
                await loading_message.edit(content="❌ You must provide a song name.")
                return

            search = self.sp.get_track(track_name, limit=1)
            if not search:
                await loading_message.edit(content="❌ Couldn't find that track.")
                return

            metadata = search[0]
            lyrics_data = self.ly.get_lyrics(metadata)

            try:
                highlighted = (
                    lyrics_data
                    if self.ly.check_instrumental(metadata)
                    else self.ly.select_lines(lyrics_data, line_range)
                )
            except Exception:
                highlighted = (
                    lyrics_data
                    if self.ly.check_instrumental(metadata)
                    else self.ly.select_lines(lyrics_data, "1-5")
                )

            # Generate the track poster and save it in the current directory
            self.ps.track(metadata, highlighted, False, "Light")

            # Find the generated PNG file in the current directory
            png_files = glob.glob(os.path.join(os.getcwd(), "*.png"))

            if not png_files:
                await loading_message.edit(
                    content="❌ Failed to generate poster or no PNG file found."
                )
                return

            # Send the first .png file found
            file_to_send = png_files[0]
            await ctx.send(file=discord.File(file_to_send))

            # Remove all .png files from the current directory
            for file in png_files:
                os.remove(file)
                print(f"Deleted {file}")

            await loading_message.edit(content="✅ Poster generated")

        except Exception as e:
            print("Error generating poster:", e)
            await loading_message.edit(
                content="⚠️ Something went wrong generating the poster."
            )


async def setup(bot):
    await bot.add_cog(BeatPrints(bot))
```

- Here I have done a bit of a workaround for sending the poster (which is a .png file) generated.
- BeatPrints doesn't return the filename of the image generated. So we scan the directory for any .png file. If any .png files are found, they are sent and then removed from the directory.
- There is also an option to choose the line numbers of the song to be added to the poster. This defaults to 1-4 but that sometimes throws an error. So I have kept 1-5 as a backup in case the former fails.
## Poster Generation
- Now is the time to test the result.
- In the server with the bot, try the command `!poster <song name>`

![generated poster](/assets/img/blog/programming/python/result.png)

## Hosting
- If we need to use the bot, we have to keep the python script running. This may not always be possible if you don't have your own server or if you can't keep PC on all the time.
- So let's upload this code to the cloud and use **railway** to keep it running.
- Railway only provides a trial version for free. You can use upto 5 dollars for free, after which you will have to pay.
- For this upload your code to a Github repo and open [railway](https://railway.com/)
- Create an account in railway and click on deploy a new project.
- Select the option - Deploy from a github repo.
- You can then select the repo with the code of the discord bot.
- Before deploying the project, make sure to add the env variables from the `.env` file to the variables section.