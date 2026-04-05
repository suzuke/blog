---
title: "AgEnD: One Telegram Bot to Run All Your AI Coding Agents"
date: 2026-04-05
draft: false
summary: "Most AI coding tools give you one agent in one terminal. AgEnD gives you a fleet -- Claude, Codex, Gemini, OpenCode, Kiro -- managed from a single Telegram chat. Here's what it does and why it exists."
description: "Most AI coding tools give you one agent in one terminal. AgEnD gives you a fleet -- Claude, Codex, Gemini, OpenCode, Kiro -- managed from a single Telegram chat. Here's what it does and why it exists."
tags: ["agend", "ai-agent", "multi-agent", "telegram", "fleet-management", "developer-tools"]
ShowToc: true
TocOpen: true
---

I've been running AI coding agents for months. Claude Code, Codex, Gemini CLI. They're good. But each one lives in its own terminal, attached to one project, and has no idea the others exist.

Three repos means three terminals. Checking progress from your phone isn't an option. Getting agents to coordinate means writing your own glue code.

I got tired of this and built [AgEnD](https://github.com/suzuke/agend). It's a daemon that manages a fleet of AI coding agents and puts the controls in a Telegram bot.

## What AgEnD is not

It's not another coding agent. It doesn't write code. It doesn't compete with Claude Code or Codex or Gemini CLI.

It's the operations layer underneath all of them.

Claude Code is a developer. AgEnD is the office. Desks, chat channels, schedules, and someone keeping track of who's working on what.

## The actual problems

Here's what running multiple AI agents looked like before AgEnD, at least for me:

- Three terminal tabs, three Claude Code sessions
- Can't check any of them from my phone
- Laptop sleeps, sessions die
- Agents have no idea the others exist
- No spending limits. I've had an agent chew through $40 overnight on a task I forgot about
- Context windows fill up silently and response quality degrades. You don't notice until the agent starts doing obviously dumb things

## Architecture

AgEnD runs as a system service (launchd on macOS, systemd on Linux). It spawns each agent in its own tmux window via node-pty, then wires everything to a Telegram bot.

```
┌─────────────────────────────────────────────┐
│                 Telegram Bot                 │
│         (one bot, multiple topics)           │
└────────────────┬────────────────────────────┘
                 │
┌────────────────▼────────────────────────────┐
│              AgEnD Daemon                    │
│  ┌──────────┬──────────┬──────────────────┐ │
│  │ Context  │   Cost   │  Hang            │ │
│  │ Guardian │  Guard   │  Detector        │ │
│  └──────────┴──────────┴──────────────────┘ │
│  ┌──────────┬──────────┬──────────────────┐ │
│  │Scheduler │  Fleet   │  MCP Server      │ │
│  │ (SQLite) │ Manager  │  (per instance)  │ │
│  └──────────┴──────────┴──────────────────┘ │
└────────────────┬────────────────────────────┘
                 │
    ┌────────────┼────────────┐
    ▼            ▼            ▼
┌────────┐ ┌────────┐ ┌────────┐
│Claude  │ │ Codex  │ │Gemini  │
│Code    │ │        │ │ CLI    │
│(tmux)  │ │(tmux)  │ │(tmux)  │
└────────┘ └────────┘ └────────┘
```

Each Telegram Forum Topic maps to one agent instance. Create a topic, bind it to a project directory, agent starts. Delete the topic, agent stops.

## Five backends, one interface

AgEnD doesn't care which coding agent you use. Currently supported:

| Backend | Auth | Notes |
|---------|------|-------|
| Claude Code | OAuth / API key | Primary, full feature support |
| OpenAI Codex | ChatGPT login / API key | Session resume supported |
| Gemini CLI | Google OAuth | |
| OpenCode | Provider config | MCP instructions not yet supported |
| Kiro CLI | AWS Builder ID | Experimental |

Run Claude Code on your backend, Codex on your frontend, Gemini on docs. All from the same Telegram chat. Switch backends when one hits rate limits or when a different model fits the task better.

## Fleet coordination

This is the part nobody else has built.

Every agent in the fleet gets an MCP server injected at startup. That server exposes fleet tools: `list_instances`, `send_to_instance`, `delegate_task`, `request_information`, `report_result`. Agents discover each other and talk directly. No central dispatcher. Peer-to-peer.

```
Agent A (backend):  "Hey frontend, I changed the API response format for /users"
Agent B (frontend): "Got it. Updating the TypeScript types and fetch calls now."
```

This happens through `send_to_instance`. Each message carries metadata: `request_kind` (query/task/report/update), `correlation_id` for linking request-response pairs, `task_summary` for visibility in Telegram.

Higher-level patterns built on top:

- `delegate_task(target, task, success_criteria)` -- assign work with expected outcomes
- `request_information(target, question, context)` -- ask another agent something
- `report_result(target, summary, artifacts, correlation_id)` -- send results back

The General Topic is a natural language dispatcher. Send it a message and it routes to the right agent, or spins up a new one via `create_instance`.

## Context rotation

Long-running sessions degrade. Context fills up, the model forgets earlier instructions, responses get worse. If you've used Claude Code for a full day you've seen this.

AgEnD's Context Guardian watches for this. When context exceeds 80% or the session hits a max age (default 8 hours), the daemon:

1. Waits for the agent to go idle
2. Captures a snapshot: recent messages, tool activity, statusline data
3. Kills the session and spawns a fresh one
4. Injects the snapshot as the first message

The snapshot is ~2000 tokens, about 0.2% of a 200K context window. The agent picks up where it left off with a clean slate.

I tried having the agent write its own handover report in earlier versions. Bad idea. Asking a degraded agent to accurately summarize its own state doesn't work. The daemon handles it externally now.

## Cost guard

Unattended agents get expensive. AgEnD tracks spending per instance and enforces daily limits.

```yaml
defaults:
  cost_guard:
    daily_limit_usd: 50
    warn_at_percentage: 80
    timezone: "Asia/Taipei"
```

Warning at 80%. Auto-pause at 100%. Reset at midnight. A daily summary posts to the General Topic with per-instance costs, restart counts, and fleet totals.

## Hang detection

Sometimes agents get stuck. Infinite loop, waiting for input that never comes, or just frozen.

The hang detector watches for 15 minutes of inactivity (configurable). When triggered, you get a Telegram notification with inline buttons: "Force restart" or "Keep waiting."

## Model failover

When your primary model (say, Opus) hits a rate limit, AgEnD can switch to a backup (Sonnet) on the next context rotation.

```yaml
instances:
  my-project:
    model: opus
    model_failover: ["sonnet"]
```

You get notified in Telegram. When rate limits clear, it switches back automatically.

## Scheduling

SQLite-backed cron schedules that survive daemon restarts.

```
You: "Every morning at 9am, check if there are open PRs that need review"
Agent: → create_schedule(cron: "0 9 * * *", message: "Check open PRs needing review")
```

Schedules are rate-limit aware. If the 5-hour API usage exceeds 85%, the trigger gets deferred instead of wasting tokens that'll get throttled.

## Permission relay

Claude Code prompts for approval on dangerous operations (`rm`, `git push`, etc.). Without AgEnD, you need to be at the terminal to approve them.

AgEnD forwards these prompts to Telegram as inline buttons. Approve or deny from your phone. There's an "Always Allow" option for tools you trust.

## How it compares

| | AgEnD | Overstory | Claude-to-IM | CrewAI / AutoGen |
|---|---|---|---|---|
| What it is | Agent fleet daemon | Multi-agent dev platform | IM bridge for Claude | General multi-agent frameworks |
| Telegram control | Native | No | Yes | No |
| Fleet coordination | P2P via MCP tools | Built-in | No | Framework-defined |
| Backend-agnostic | 5 backends | Own agents | Claude only | LLM-agnostic but not CLI agents |
| Persistence | System service + SQLite | Cloud | Varies | In-memory |
| Cost control | Daily limits per instance | Platform-managed | No | No |
| Context management | Auto-rotation | N/A | No | No |
| Target user | Dev running CLI agents | Teams adopting AI dev | Individual Claude users | AI app builders |

CrewAI and AutoGen are frameworks for building multi-agent applications from scratch. AgEnD doesn't ask you to build anything. You point it at the coding agents you already use and it handles the rest.

## Getting started

```bash
# Prerequisites
brew install tmux    # macOS

# Install
npm install -g @suzuke/agend

# Interactive setup
agend init

# Install as system service and start
agend install --activate
```

`agend init` walks you through the Telegram bot setup, configuring your first instance, and picking a backend. Config goes in `~/.agend/fleet.yaml`.

Basic operations:

```bash
agend fleet start          # Start all instances
agend fleet stop           # Stop all instances
agend fleet status         # Show fleet status
agend fleet logs my-proj   # Tail instance logs
```

Once running, open Telegram, create a topic in your group, and start talking to your agent.

## What's next

AgEnD is at v1.11. The core loop (spawn agents, manage from Telegram, let them collaborate) is stable. Current work:

- More channel adapters. Discord is at MVP. Community adapters welcome.
- Better cross-backend compatibility, especially OpenCode and Kiro MCP instructions support.
- Team-level coordination patterns.
- Webhook integrations for external monitoring.

Repo is at [github.com/suzuke/agend](https://github.com/suzuke/agend). MIT licensed. If you're running multiple coding agents already and managing them by hand, this might save you some headaches.
