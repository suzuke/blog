---
title: "My AI agents burned tokens all night doing nothing"
date: 2026-03-20
draft: false
summary: "I left OpenFang running overnight. 170 LLM calls later, 80% of them were the agent saying 'nothing to do.' Here's the bug I found in the scheduling code."
description: "I left OpenFang running overnight. 170 LLM calls later, 80% of them were the agent saying 'nothing to do.' Here's the bug I found in the scheduling code."
tags: ["ai", "agent", "openfang", "debugging", "rust"]
ShowToc: true
TocOpen: true
---

Something felt off when I checked my Claude Code usage this week. One day had 809 sessions. The usual is around 100.

I run [OpenFang](https://github.com/RightNow-AI/openfang), an open-source agent OS, as a background daemon on my laptop. I'd activated the built-in Researcher Hand through the dashboard a few days ago, used it once, then forgot about it.

It didn't forget about me.

## The numbers

I dug into the session logs stored under `~/.claude/projects/`. 497 session files for the openfang project, almost all from a single day.

After parsing through the JSONL, the picture was clear:

| Category | Sessions | Data |
|---|---|---|
| Code Reviewer (superpowers plugin) | 346 | 22.9M |
| Researcher Hand (openfang) | 170 | 19.7M |
| Conversation Summarizer | 21 | 0.3M |
| Other | 7 | 2.8M |

The 346 code reviewer sessions were from the superpowers plugin doing its thing during development. Fine. Expected.

The 170 Researcher Hand sessions were not expected.

## Every 60 seconds, like clockwork

The timestamps told the story:

```
2026-03-19T13:06:57
2026-03-19T13:07:57
2026-03-19T13:08:57
2026-03-19T13:09:57
2026-03-19T13:10:57
...
```

One session per minute. For 12 hours straight.

Each session started with the same message:

```
[AUTONOMOUS TICK] You are running in continuous mode.
Check your goals, review shared memory for pending tasks,
and take any necessary actions.
```

And the response, 80% of the time:

```
NO_REPLY
```

The agent woke up, looked around, found nothing to do, and went back to sleep. But "sleep" still meant a full LLM round-trip: system prompt, tool definitions, the whole context window. Every 60 seconds.

The final breakdown:
- 169/170 sessions had the `[AUTONOMOUS TICK]` message
- 136/170 (80%) responded with `NO_REPLY`
- 35/170 (20%) actually did useful work
- One session ended with "You've hit your limit -- resets 3am"

## Finding the bug

OpenFang has a concept called "Hands" -- bundled agent extensions with their own system prompts, tools, and settings. The Researcher Hand is one of them. Its config file sets `max_iterations = 80`, which is supposed to cap how many steps the agent can take in a single conversation.

But in `kernel.rs`, there's this:

```rust
schedule: if def.agent.max_iterations.is_some() {
    ScheduleMode::Continuous {
        check_interval_secs: 60,
    }
} else {
    ScheduleMode::default() // Reactive
},
```

If `max_iterations` exists, the Hand gets put into Continuous mode with a 60-second tick. The code assumes "this agent can take multiple steps" means "this agent should run autonomously in a loop." Those are completely different things.

8 out of 30 bundled Hands set `max_iterations`: browser, clip, collector, lead, predictor, researcher, trader, twitter. Activate any of them, and they start ticking every minute whether they have work or not.

Worse: activated Hands get persisted to `~/.openfang/hand_state.json`. So every time the daemon restarts, they come right back. I activated Researcher Hand once through the dashboard, forgot about it, and it's been ticking ever since.

## What needs to change

The `max_iterations` field and the scheduling mode need to be decoupled. Something like an explicit `autonomous = true` in the Hand's TOML file for Hands that actually want to run in a loop.

And idle ticks should back off. If the agent says `NO_REPLY` three times in a row, maybe wait 5 minutes instead of 60 seconds. Then 15. Then 30. There's no reason to keep asking every minute when there's obviously nothing happening.

I filed [issue #756](https://github.com/RightNow-AI/openfang/issues/756).

## Takeaway

If you run OpenFang and have activated any Hands through the dashboard, check `~/.openfang/hand_state.json`. If there's anything in there you're not actively using, remove it and restart the daemon. Otherwise those agents are waking up every minute, sending LLM calls into the void.

I set up an agent to research things for me. The biggest thing it researched was how much it costs to have an agent sit around doing nothing.
