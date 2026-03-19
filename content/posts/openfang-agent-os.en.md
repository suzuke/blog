---
title: "OpenFang: Running a Full AI Agent OS on My Laptop"
date: 2026-03-19
draft: false
summary: "An open-source Agent OS written in Rust. 14 crates, 170K lines, 42 communication channels. I installed it, hooked up Telegram, and let Claude run in the background."
description: "An open-source Agent OS written in Rust. 14 crates, 170K lines, 42 communication channels. I installed it, hooked up Telegram, and let Claude run in the background."
tags: ["ai", "agent", "rust", "openfang", "self-hosting"]
ShowToc: true
TocOpen: true
---

Last week I found a project on GitHub called [OpenFang](https://github.com/RightNow-AI/openfang). The README calls it an "open-source Agent Operating System."

I usually skip projects with titles like that. But I clicked into the Cargo.toml. 14 crates, clean workspace layout. Not a toy.

Spent an evening getting it running. Here are my notes.

## What it actually is

OpenFang is a multi-agent platform written in Rust. If Claude Code is your AI pair programmer, OpenFang is the system that lets a bunch of AI agents run on their own in the background.

The architecture:

```
CLI / Web Dashboard
       │
    API Layer (Axum HTTP + WebSocket)
       │
    Kernel (scheduling, metering, events, workflows)
       │
    Runtime (agent loop, LLM drivers, MCP, sandbox)
       │
    ┌───┴───┐
  Memory  Channels (42 platforms)
```

14 crates. 170K lines of Rust. 2,200 tests, all green. Zero clippy warnings. Cleaner than a lot of commercial codebases I've worked with.

## Installation

Clone, build, five minutes.

```bash
git clone https://github.com/RightNow-AI/openfang
cd openfang
cargo build --release -p openfang-cli
./target/release/openfang init --quick
```

During init it detected Ollama running on my machine and auto-configured itself. No API key needed.

I wanted Claude though, so I changed the config:

```toml
[default_model]
provider = "claude-code"
model = "claude-code/sonnet"
api_key_env = ""
```

It can piggyback on your Claude Code subscription. Under the hood it spawns `claude -p` as a subprocess, and authentication goes through Claude Code itself. No extra cost.

## 30 agent templates

Ships with 30 built-in agents: coder, researcher, debugger, security-auditor, orchestrator, and more.

What makes them different? The system prompt defines their expertise. Capabilities control which tools they can touch. Resources set token limits. All in TOML:

```toml
name = "coder"
[model]
temperature = 0.3
system_prompt = "You are Coder, an expert software engineer..."

[capabilities]
tools = ["file_read", "file_write", "shell_exec"]
shell = ["cargo *", "git *", "npm *"]

[resources]
max_llm_tokens_per_hour = 200000
```

The coder can run shell commands. The researcher can only search the web. Permissions are granular.

The orchestrator is the fun one. It doesn't do work itself. It analyzes the task, then delegates to coder, researcher, and other specialists. AI tech lead, basically.

## 42 communication channels

This I didn't expect. Bidirectional bridges to 42 messaging platforms:

Telegram, Discord, Slack, WhatsApp, LINE, Signal, Teams, Matrix, Email, Mastodon, Bluesky, Reddit, IRC. The list keeps going.

Hooking up Telegram took 30 seconds. One line in the config, restart.

```toml
[channels.telegram]
bot_token_env = "TELEGRAM_BOT_TOKEN"
```

The logs:

```
INFO Telegram bot @OpenFang789_Bot connected
INFO Telegram: cleared webhook, polling mode active
INFO telegram channel bridge started
```

And then I was chatting with Claude Sonnet on my phone. Through Telegram. That was a weird moment.

## How it differs from Claude Code

Different layer entirely:

| | Claude Code | OpenFang |
|---|---|---|
| What it is | AI that helps you write code | OS that manages AI agent clusters |
| Agents | 1 | Many, each with its own identity and permissions |
| Communication | Terminal | 42 platforms |
| Scheduling | None | Built-in cron |
| Cost tracking | None | Per-agent budget tracking |

I use Claude Code to develop and explore OpenFang. I use OpenFang to deploy and run agents. One writes, the other runs.

## Security

A few things I noticed in the code:

- Binds to `127.0.0.1` by default, local only
- Agent tool calls go through a capability whitelist
- API keys use `Zeroizing<String>`, memory zeroed on drop
- Ed25519 signatures, HMAC auth, Merkle hash chain audit trail
- WASM sandbox with fuel limits and epoch interrupts

It's v0.4.9 though. Early. Don't run anything sensitive on it.

## Maturity

The core engine (runtime, kernel, API, CLI, channels) is production-quality. It works. It's stable.

The rest is still growing. Memory, skills, WASM extensions, P2P networking are all early-stage. Right now it's a solid multi-agent orchestration platform. The full "Agent OS" vision will take more time.

## Worth installing?

If you just want to chat with AI, you don't need this. ChatGPT and Claude are fine.

But if you want agents running in the background, doing scheduled tasks, listening on Telegram, chaining together to handle real work, this is the most complete open-source option I've found.

Try it. Worst case, `rm -rf ~/.openfang` and it's gone.
