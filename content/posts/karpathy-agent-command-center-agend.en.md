---
title: "Karpathy Wants an Agent Command Center. I Already Built One."
date: 2026-04-05
draft: false
summary: "Karpathy ran 8 agents in tmux and called it 'a mess.' He wants an agent command center IDE. AgEnD has been solving these exact problems since January."
description: "Karpathy ran 8 agents in tmux and called it 'a mess.' He wants an agent command center IDE. AgEnD has been solving these exact problems since January."
tags: ["agend", "karpathy", "ai-agent", "multi-agent", "agent-command-center", "developer-tools"]
ShowToc: true
TocOpen: true
---

Andrej Karpathy has been running agents. A lot of them. 8 at once -- 4 Claude, 4 Codex -- each on its own GPU running nanochat experiments. His verdict:

> "it doesn't work and it's a mess"

A few days later, he posted what he actually wants:

> "tmux grids are awesome, but I feel a need to have a proper 'agent command center' IDE for teams of them. I want to see/hide toggle them, see if any are idle, pop open related tools (e.g. terminal), stats (usage), etc."

And the bigger picture:

> "the basic unit of interest is not one file but one agent. It's still programming."

I read these and thought: I've been building this for months. Not as an IDE, but as a daemon with a Telegram interface. Different form factor, same problems.

Here's what Karpathy described as pain points, and how [AgEnD](https://github.com/suzuke/agend) handles each one.

## 1. Staring at tmux grids

Karpathy's setup is a tmux grid showing all agent sessions. You can see their terminals, watch them work. The problem: you have to be at your computer.

He runs overnight experiments. He sleeps while agents work. But if something goes wrong at 3am, he won't know until morning.

AgEnD puts the entire control surface in Telegram. Each agent gets its own Forum Topic. Messages flow in real time. I check my agents from bed, from a cafe, from the bathroom. When an agent finishes a task or hits an error, I get a notification on my phone.

The tmux sessions still exist -- AgEnD runs each agent inside tmux via node-pty. But you don't need to look at them. Telegram is your window.

## 2. Detecting idle agents by eye

> "I want to see if any are idle"

Karpathy wants idle detection. Right now he's eyeballing tmux panes to see which agents have stopped producing output.

AgEnD's Hang Detector watches for inactivity. Default is 15 minutes, configurable. When triggered, you get a Telegram notification with two buttons: "Force restart" or "Keep waiting." Topic icons update to show running vs stopped instances at a glance.

You stop checking. It checks for you.

## 3. Watcher scripts as glue code

Karpathy mentions "building watcher scripts to keep them looping." These are his own scripts, not open-sourced, custom glue that monitors agents and restarts them when they stop.

AgEnD replaces all of that:

- Crash recovery with configurable restart policy (max retries, exponential backoff)
- Error pattern detection per backend (rate limits, auth failures, usage limits)
- Model failover -- when Opus hits a rate limit, auto-switch to Sonnet
- Context rotation -- when context fills up past 80%, daemon captures a snapshot, kills the session, starts a fresh one, injects the snapshot

All built in. You don't write watcher scripts, you configure a YAML file.

## 4. Agent coordination is "a mess"

Karpathy tried 8 agents collaborating. His coordination: git branches for each research program, git worktrees for isolation, simple files for communication. He called it a mess.

AgEnD's fleet coordination uses MCP tools injected into every agent at startup:

- `send_to_instance` -- direct peer-to-peer messaging with structured metadata
- `delegate_task` -- assign work with success criteria
- `request_information` -- ask another agent something
- `report_result` -- return results with correlation IDs

No central dispatcher. Each agent discovers the others via `list_instances` and communicates directly. The General Topic routes natural-language requests to the right agent automatically.

It's not magic -- agents still make bad decisions sometimes. But at least the communication channel isn't "simple files."

## 5. No unified usage/cost tracking

> "stats (usage)"

Karpathy wants usage stats. When you're running 8 agents, costs add up fast and there's no single place to see the total.

AgEnD's Cost Guard tracks spending per instance with daily limits:

```yaml
defaults:
  cost_guard:
    daily_limit_usd: 50
    warn_at_percentage: 80
```

Warning at 80%. Auto-pause at 100%. A daily summary posts to the General Topic showing per-instance costs, restart counts, and fleet totals. You know exactly how much each agent burned before you wake up.

## 6. Results scattered across git branches

Karpathy's autoresearch uses one branch per experiment. Results end up scattered across branches and you have to manually check which ones had improvements.

AgEnD doesn't replace git branching -- that's still how agents isolate their work. But `report_result` gives structure to how results flow back. When Agent B finishes a task delegated by Agent A, it sends a structured report with summary, artifacts, and correlation_id linking it to the original request. These show up in Telegram where you can see them in context.

Shared Decisions (via `post_decision` and `list_decisions`) let agents record architectural decisions that persist across the fleet. Not git history -- more like an ADR log that every agent can read.

## 7. Multi-backend switching

Karpathy runs both Claude and Codex. Different models for different tasks. Switching between them means different CLIs, different auth, different quirks.

AgEnD abstracts this. Five backends behind one interface:

- Claude Code (OAuth / API key)
- OpenAI Codex (ChatGPT login / API key)
- Gemini CLI (Google OAuth)
- OpenCode (provider config)
- Kiro CLI (AWS Builder ID)

Each instance picks its backend in `fleet.yaml`. Run Claude on your backend, Codex on your frontend, Gemini on docs. Switch when one hits rate limits. The Telegram interface is the same regardless.

## 8. Knowledge doesn't persist

Chat history vanishes when sessions end. Context fills up and gets rotated. The agent loses what it learned.

AgEnD's Context Guardian captures a snapshot before rotation -- recent messages, tool activity, statusline data (~2000 tokens). The new session starts with this snapshot injected. Not perfect, but the agent doesn't start completely blank.

Shared Decisions help too. When an agent makes an architectural choice, it can record it via `post_decision`. Other agents (and future sessions of the same agent) can read these via `list_decisions`. Knowledge outlives individual sessions.

## Different form factor, same problem

Karpathy wants an IDE. A visual command center with panels, toggles, stats. AgEnD is a daemon with a Telegram bot. Different shape, same underlying architecture:

| What Karpathy wants | AgEnD equivalent |
|---|---|
| Visual agent panels | Telegram Forum Topics |
| Idle detection | Hang Detector + topic icons |
| Watcher scripts | Built-in crash recovery + failover |
| Agent coordination | MCP fleet tools (P2P) |
| Usage stats | Cost Guard + Daily Summary |
| Results tracking | report_result + Shared Decisions |
| Multi-backend | 5 backend abstractions |
| Knowledge persistence | Context rotation snapshots + Shared Decisions |

The form factor question is real. Some people want a desktop app with drag-and-drop panels. I wanted something I could use from my phone at 2am. Both are valid.

But the underlying problems are the same, and they're solved at the daemon level, not the UI level. AgEnD could have a desktop frontend tomorrow and the core wouldn't change. The fleet manager, cost guard, hang detector, context rotation -- all of that is backend logic. Telegram is just one adapter.

## What Karpathy got right

The most useful thing Karpathy said wasn't about tooling:

> "the basic unit of interest is not one file but one agent. It's still programming."

This is exactly right. We're not past the age of IDEs. We need new IDEs for a new unit of work. The old IDE helped you manage files. The new one helps you manage agents.

AgEnD is one answer. There will be others. But if you're currently managing agents via tmux grids and watcher scripts, you don't have to wait for the perfect IDE. The problems are solvable now.

Repo: [github.com/suzuke/agend](https://github.com/suzuke/agend). MIT licensed.
