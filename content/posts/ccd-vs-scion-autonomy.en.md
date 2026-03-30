---
title: "The next step in multi-agent: cross-system collaboration to overcome model bias"
date: 2026-03-30
draft: false
summary: "CCD and Scion have nearly identical architectures, but Scion already runs mixed LLM systems. That's not just a feature difference — different models carry different training biases, and mixing them is the only way to get real diversity of thought."
description: "CCD and Scion have nearly identical architectures, but Scion already runs mixed LLM systems. That's not just a feature difference — different models carry different training biases, and mixing them is the only way to get real diversity of thought."
tags: ["multi-agent", "agent", "claude-code", "open-source", "developer-tools", "gemini"]
ShowToc: true
TocOpen: true
---

CCD and [Scion](https://github.com/GoogleCloudPlatform/scion) solve nearly the same problem: run multiple AI agents in parallel, let them coordinate, get work done that one agent couldn't handle alone. Their underlying architectures are also similar — both use tmux to manage sessions, both give each agent an isolated workspace.

But there's one difference that points to something important about where multi-agent collaboration actually needs to go.

## The structural similarity

Worth noting first: these two tools are the same kind of thing.

- Multiple AI agents running simultaneously
- Each agent has its own isolated git worktree
- Agents can coordinate with each other
- Both use tmux to manage sessions

Scion uses containers (Docker/Podman/K8s) for isolation; CCD uses processes. That's an implementation difference. The problem they're solving is the same: how do you run multiple agents concurrently without them stepping on each other?

## The real difference: what's running inside

CCD currently supports only Claude Code as its underlying engine. Every instance is a Claude Code process. Ten agents means ten Claudes.

Scion is **harness-agnostic**. It runs Gemini CLI, Claude Code, Codex, OpenCode, or anything that can execute inside a container. Each agent container has its own home directory, credentials, and config — fully independent. You can have a Claude agent and a Gemini agent looking at the same code simultaneously.

This isn't just "supports more tools." It's a fundamentally different capability.

## Why cross-system collaboration matters

Every LLM carries its training biases.

Claude has Claude's tendencies — shaped by its training data, RLHF direction, and architecture. Gemini has Gemini's. Codex has Codex's. This isn't about which is better. It's about the fact that each model's training history shapes *how it tends to see problems*.

Running multiple Claude agents in parallel gets you multiple perspectives carrying the same biases. They can make the same systematic errors together and reinforce each other. Three Claudes doing code review will all miss the same things if Claude has a blind spot in that area.

Cross-system collaboration breaks this. What Claude doesn't see, Gemini might. The solution Gemini defaults to, Claude might challenge. Different training backgrounds produce genuinely different perspectives — not just more compute.

That's the real value of multi-agent. Not parallelization. Diversity.

## How Scion approaches this

Scion's design philosophy is "less is more": no rigid orchestration patterns, just give each agent a set of tools and let the models figure out how to coordinate.

Because it's harness-agnostic, a Gemini agent and a Claude agent can use the same toolset to communicate, each reasoning with its own language model, working within the same task framework. The coordination logic emerges from the models themselves rather than being hardcoded.

This opens up an interesting experimental question: when agents from different systems collaborate without being told how, what patterns emerge?

## CCD's current state and the next step

CCD is currently an orchestration layer for Claude Code. It does that well — manage multiple instances from your phone, persist schedules, auto-rotate context, P2P cross-instance messaging.

But being tied to a single LLM system has a fundamental ceiling: all agents carry the same training biases.

Going harness-agnostic is the obvious next direction. Same tmux-based architecture, but each instance can be a different CLI agent — not just Claude Code, but Gemini CLI, Codex, or others. The interface (Telegram bot) stays the same. Scheduling and coordination stay the same. The agents just become more diverse.

Then you can direct a genuinely varied AI development team from your phone: one Claude handling architecture, one Gemini doing security review, one Codex focused on optimization — each bringing a different training perspective to the same problem, checking each other's blind spots.

---

The first version of multi-agent was "more agents." The next version is "diverse agents." Scion is already heading there.

---

Related:
- [claude-channel-daemon (CCD)](https://github.com/suzuke/claude-channel-daemon)
- [Scion (Google Cloud Platform)](https://github.com/GoogleCloudPlatform/scion)
- [Scion agents.md](https://github.com/GoogleCloudPlatform/scion/blob/main/agents.md)
