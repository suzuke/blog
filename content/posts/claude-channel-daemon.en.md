---
title: "Keeping Claude Code Always On: I Built a Daemon to Babysit It"
date: 2026-03-21
draft: false
summary: "Claude Code's Telegram plugin dies when you close the terminal. I wrote a daemon to keep it alive, with automatic context rotation, remote tool approval, and voice transcription."
description: "Claude Code's Telegram plugin dies when you close the terminal. I wrote a daemon to keep it alive, with automatic context rotation, remote tool approval, and voice transcription."
tags: ["claude-code", "telegram", "daemon", "ai-agent", "node-pty"]
ShowToc: true
TocOpen: true
---

Yesterday I wrote a setup guide for controlling Claude Code via Telegram. Everything worked great — I could lie on the couch, pull out my phone, and tell the agent to edit code, run tests, look things up.

Didn't take long for the cracks to show.

## You Can't Close the Terminal

The Telegram plugin is an MCP server attached to a Claude Code CLI session. Session alive, bot alive. Session dead, bot dead.

Translation: your terminal has to stay open forever.

Close the laptop lid? Dead. SSH times out? Dead. Accidentally hit Ctrl+C? Dead. macOS decides to update? Dead.

Way too fragile. I wanted something like PostgreSQL — starts on boot, restarts on crash, keeps running whether or not I'm at my desk.

## Context Fills Up

Even if you keep the terminal open, there's the context window problem.

The longer you use a session, the more context it eats. Read a few dozen files, run a bunch of commands, and you're bumping against the ceiling. Once it gets full, Claude noticeably degrades — slower responses, forgets things you told it earlier, worse judgment.

I came across a study suggesting agents perform best when context stays below 40%. Past that, quality slides.

So beyond keeping the session alive, I also needed it to automatically swap in a fresh session before context got too full.

## Remote Approval

Third problem: permissions.

Claude Code pops a terminal prompt when it wants to run something dangerous — `rm`, `git push`, that sort of thing. But if I'm out and only have Telegram on my phone, I can't see the terminal.

Auto-approve everything? Too risky. One bad call from the agent and `rm -rf` wipes your project.

I needed a way to forward those approval prompts to Telegram. Tap a button on my phone, done.

## What I Built

Spent a few hours writing [claude-channel-daemon](https://github.com/suzuke/claude-channel-daemon). A Node.js program that spawns Claude Code CLI inside a pseudo-terminal via `node-pty`.

Four pieces:

**Process Manager** — Launches Claude Code, handles session resume, auto-restarts on crash. Restarts use exponential backoff (1s, 2s, 4s, 8s… capped at 60s). Counter resets after five minutes of stable uptime.

**Context Guardian** — Reads Claude's status line JSON every two seconds to check context usage. When it crosses the threshold, it kills the old session, clears the session ID, and starts a fresh one. I set the threshold at 40%.

**Memory Layer** — Watches Claude's memory directory with chokidar. Any change gets backed up to SQLite. Sessions rotate, memories don't.

**Service Installer** — Generates macOS launchd or Linux systemd service files. Starts on boot.

## Where the Approval System Lives

Took a wrong turn here.

Initially I built an HTTP server inside the daemon to handle approvals. Got it working, then realized the problem — the Telegram plugin (grammy) is already long-polling `getUpdates`. A second poller from the daemon triggers 409 conflicts. Telegram only allows one consumer.

Ended up putting it in the plugin instead. The plugin already has the bot instance and `callback_query` handler, so it's the natural place.

Implementation: a `Bun.serve` HTTP server inside the plugin, listening on `127.0.0.1:18321`. Claude Code's PreToolUse hook POSTs tool invocation details to this endpoint on every call.

The server checks if it's dangerous — regex patterns for `rm`, `sudo`, `git push --force`, plus sensitive paths like `.env` and `.claude/settings.json`. Safe operations pass through silently. Dangerous ones get forwarded to Telegram with two inline buttons: ✅ Approve and ❌ Deny.

Press a button, the hook gets its response, Claude proceeds or stops. No response in two minutes means automatic denial.

## Voice Transcription

The stock Telegram plugin just passes a file_id for voice messages — no transcription. But I use voice input most of the time, so I added Groq's Whisper API (`whisper-large-v3-turbo`).

Straightforward pipeline: receive voice → download OGG → hit Groq API → get text → prepend `[voice message transcription]` and feed it to Claude. Chinese recognition is decent. Homophones trip it up occasionally, but it gets the meaning right most of the time.

## Maintaining a Modified Plugin

Once you modify the official plugin, you have to think about upstream updates.

I forked [claude-plugins-official](https://github.com/anthropics/claude-plugins-official) and split the two features into separate branches:

- `feat/telegram-voice-transcription` — voice transcription
- `feat/telegram-remote-approval` — remote approval system

Then added a GitHub Actions workflow that runs daily:

1. Fetch upstream
2. Merge into my main
3. Merge main into both feature branches
4. Combine both into a `deploy` branch
5. Open an issue if there's a conflict

Deploying is one `cp` — copy `server.ts` from the deploy branch to `~/.claude/plugins/cache/`.

## How's It Running

Just got it up. So far so good. Session rotation is smooth — after a swap, Claude's memory system loads automatically and it still remembers prior conversations. Remote approval works well too, just tap the notification on my phone.

The 40% context threshold needs more observation time, but response quality and speed feel solid so far.

Code is on [GitHub](https://github.com/suzuke/claude-channel-daemon), MIT licensed. If you're using Claude Code's Telegram channel, might be worth a look.
