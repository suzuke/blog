---
title: "How I talk to Claude Code from my phone via Telegram"
date: 2026-03-20
draft: false
summary: "Claude Code supports Telegram as a channel now. Set up a bot, pair it, and you can message your agent from your phone."
description: "Claude Code supports Telegram as a channel now. Set up a bot, pair it, and you can message your agent from your phone."
tags: ["claude-code", "telegram", "ai-agent", "setup-guide"]
ShowToc: true
TocOpen: true
---

I spend most of my day in a terminal running Claude Code. But sometimes I'm on the couch or out somewhere and I want to tell my agent to do something. I used to SSH in from my phone. It works, but typing in a tiny terminal is miserable.

I'd been looking into this for a while. Tried a bunch of third-party tools built around Claude, and even started writing my own MCP server to bridge Telegram. Then I found out Claude Code ships this officially now, and the approach is basically the same as what I was building — an MCP server as the middleware between Telegram and the session. Saved me from reinventing the wheel.

You wire up a Telegram bot to your running session, and messages you send the bot get forwarded to your agent. Replies show up in the chat. Basically texting your terminal.

Took me about five minutes to set up.

## What you need

- Claude Code v2.1.80 or later
- [Bun](https://bun.sh) runtime (the plugin server needs it)
- A Telegram account

## Step 1: Create a Telegram bot

Open Telegram, message [@BotFather](https://t.me/BotFather), send `/newbot`. It asks for a display name and a username (has to end in `bot`). You get back a token like `123456789:AAHfiqksKZ8...`. Copy the whole thing.

## Step 2: Install the plugin

In a Claude Code session:

```
/plugin install telegram@claude-plugins-official
```

Done.

## Step 3: Save the bot token

```
/telegram:configure YOUR_TOKEN_HERE
```

This writes the token to `~/.claude/channels/telegram/.env`. If you'd rather not paste a token into a session, just edit the file directly.

## Step 4: Restart with the channel flag

Exit and restart:

```bash
claude --channels plugin:telegram@claude-plugins-official
```

The Telegram MCP server starts up and connects to the Bot API. You should see it in the plugin list.

## Step 5: Pair your identity

DM your bot on Telegram. It replies with a 6-character code like `a4f91c`. Back in Claude Code:

```
/telegram:access pair a4f91c
```

Your Telegram user ID goes into the allowlist. From here on, messages you send the bot reach Claude.

## Step 6: Lock it down

By default the bot is in `pairing` mode, which means any random person who DMs it gets a pairing code. You don't want that sitting open. Once you've paired everyone who needs access:

```
/telegram:access policy allowlist
```

Strangers get nothing. No error, no code, just silence.

## What it looks like in practice

I text the bot "check if the deploy succeeded" from my phone. Claude runs `gh run list` and replies in Telegram with the result. Photos work too; the bot downloads them and Claude can see them.

The bot can do three things: reply with text and file attachments, react to your messages with emoji (I use this as an "acknowledged" signal), and edit messages it already sent (handy for replacing "working..." with actual results).

## Things worth knowing

The MCP server runs locally on your machine. Nothing goes to a third party besides Telegram's own API for message delivery.

There's no message history. The Bot API doesn't expose past messages, so the bot only sees what arrives in real time. If you need to reference something earlier, paste it in.

Token changes need a restart (or `/reload-plugins`). Access changes take effect immediately since `access.json` gets re-read on every incoming message.

## Managing access

```bash
# Check current state
/telegram:access

# Add someone by their numeric Telegram ID
/telegram:access allow 412587349

# Remove someone
/telegram:access remove 412587349

# Approve a pending pairing
/telegram:access pair <code>

# Switch between policies
/telegram:access policy pairing    # temporary: for onboarding new users
/telegram:access policy allowlist  # permanent: locked to known users
/telegram:access policy disabled   # kill switch: drops everything
```

To find someone's numeric Telegram ID, have them message [@userinfobot](https://t.me/userinfobot).

## Why not just SSH?

I did the phone-SSH thing for months. The terminal is too small, session management (tmux, mosh) adds friction, and copy-pasting output is awful.

Telegram gives me a proper chat interface with images and code blocks, and the client keeps history. No more fiddling with a tiny SSH terminal on my phone — I just text the bot. The Claude Code session on my computer still needs to be running, though, or the bot goes silent. But since Telegram messages land in that same session, all your installed skills, memory, and plugins are there. It's the same experience as sitting at your desk.
