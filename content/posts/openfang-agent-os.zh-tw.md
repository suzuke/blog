---
title: "OpenFang：我在本機跑了一整個 AI Agent 作業系統"
date: 2026-03-19
draft: false
summary: "一個用 Rust 寫的開源 Agent OS，14 個 crate、17 萬行程式碼、42 個通訊頻道。我把它裝起來，接上 Telegram，讓 Claude 在背景自己跑。"
description: "一個用 Rust 寫的開源 Agent OS，14 個 crate、17 萬行程式碼、42 個通訊頻道。我把它裝起來，接上 Telegram，讓 Claude 在背景自己跑。"
tags: ["ai", "agent", "rust", "openfang", "self-hosting"]
ShowToc: true
TocOpen: true
---

上週在 GitHub 看到一個專案叫 [OpenFang](https://github.com/RightNow-AI/openfang)。README 寫得很狂——「開源 Agent 作業系統」。

這種標題我通常直接跳過。但我點進去看了一下 Cargo.toml，14 個 crate，workspace 整整齊齊。不是玩具。

花了一個晚上把它跑起來。以下是筆記。

## 它到底是什麼

OpenFang 是用 Rust 寫的 multi-agent 平台。如果 Claude Code 是你的 AI 配對程式設計師，OpenFang 就是讓一群 AI agent 自己在背景跑的管理系統。

架構長這樣：

```
CLI / Web Dashboard
       │
    API 層 (Axum HTTP + WebSocket)
       │
    Kernel (排程、計費、事件、工作流)
       │
    Runtime (Agent 迴圈、LLM 驅動、MCP、沙箱)
       │
    ┌───┴───┐
  Memory  Channels (42 個通訊平台)
```

14 個 crate，17 萬行 Rust，2200 個測試全綠，clippy 零警告。比很多商業專案乾淨。

## 安裝

比想像中簡單。clone 下來，build，五分鐘。

```bash
git clone https://github.com/RightNow-AI/openfang
cd openfang
cargo build --release -p openfang-cli
./target/release/openfang init --quick
```

init 的時候它自動偵測到我本機有跑 Ollama，直接設好了。不用填任何 API key。

但我想用 Claude，所以改了 config：

```toml
[default_model]
provider = "claude-code"
model = "claude-code/sonnet"
api_key_env = ""
```

它可以直接吃 Claude Code 的訂閱。原理是 spawn `claude -p` 當 subprocess，認證走 Claude Code 本身。不用另外付錢。

## 30 個 Agent 模板

內建 30 個 agent：coder、researcher、debugger、security-auditor、orchestrator 等等。

差異在哪？三件事：system prompt 決定它的專業、capabilities 決定它能碰什麼工具、resources 決定 token 上限。全部用 TOML 寫：

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

coder 能跑 shell，researcher 只能搜尋。權限切得很細。

最好玩的是 orchestrator——它自己不動手，分析完任務後把工作派給 coder、researcher。AI 版的 tech lead。

## 42 個通訊頻道

這個我沒預期到。42 個通訊平台的雙向橋接：

Telegram、Discord、Slack、WhatsApp、LINE、Signal、Teams、Matrix、Email、Mastodon、Bluesky、Reddit、IRC……想得到的幾乎都有。

接 Telegram 花了 30 秒。config 加一行，重啟。

```toml
[channels.telegram]
bot_token_env = "TELEGRAM_BOT_TOKEN"
```

日誌：

```
INFO Telegram bot @OpenFang789_Bot connected
INFO Telegram: cleared webhook, polling mode active
INFO telegram channel bridge started
```

然後我就在手機上跟 Claude Sonnet 聊天了。透過 Telegram。有點超現實。

## 跟 Claude Code 差在哪

不同層級的東西：

| | Claude Code | OpenFang |
|---|---|---|
| 本質 | 幫人寫 code 的 AI | 管理 AI agent 叢集的 OS |
| Agent 數量 | 1 個 | 多個，各有獨立身份和權限 |
| 通訊 | 終端機 | 42 個平台 |
| 排程 | 無 | 內建 cron |
| 計費 | 無 | per-agent 預算追蹤 |

可以互補。我現在用 Claude Code 開發和探索 OpenFang，用 OpenFang 部署和運行 agent。一個寫，一個跑。

## 安全性

幾個我有注意到的設計：

- 預設綁定 `127.0.0.1`，只有本機能連
- Agent 工具呼叫走 capability 白名單
- API key 用 `Zeroizing<String>`，drop 時記憶體歸零
- Ed25519 簽章、HMAC 認證、Merkle hash chain 審計
- WASM 沙箱有 fuel 限制和 epoch 中斷

但它現在是 v0.4.9。早期。別拿來跑敏感的東西。

## 完成度

核心引擎（runtime、kernel、API、CLI、channels）是生產級品質。能用，穩定。

周邊的東西還在長——memory、skills、WASM extensions、P2P 網路都還早期。現在它更像一個很強的 multi-agent orchestration platform，離「Agent OS」的完整願景還有距離。

## 值得裝嗎

如果你只是想跟 AI 聊天，不需要。ChatGPT 跟 Claude 就夠了。

但如果你想讓 agent 在背景自己跑——定時任務、監聽 Telegram、多個 agent 串起來做事——這是我目前看到最完整的開源選項。

裝起來試試。不喜歡的話 `rm -rf ~/.openfang` 就沒了。
