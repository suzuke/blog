---
title: "AgEnD：用一個 Telegram Bot 管理所有 AI Coding Agent"
date: 2026-04-05
draft: false
summary: "多數 AI coding 工具給你一個 agent、一個終端機。AgEnD 給你一支艦隊 -- Claude、Codex、Gemini、OpenCode、Kiro -- 全部在一個 Telegram 聊天室裡操控。"
description: "多數 AI coding 工具給你一個 agent、一個終端機。AgEnD 給你一支艦隊 -- Claude、Codex、Gemini、OpenCode、Kiro -- 全部在一個 Telegram 聊天室裡操控。"
tags: ["agend", "ai-agent", "multi-agent", "telegram", "fleet-management", "developer-tools"]
ShowToc: true
TocOpen: true
---

用 AI coding agent 好幾個月了。Claude Code、Codex、Gemini CLI 都不錯，但都一樣：一個 agent、一個終端機、一個專案。

三個 repo 要同時跑？開三個終端機。手機看進度？沒辦法。讓 agent 之間互通？自己寫膠水。

搞煩了，寫了 [AgEnD](https://github.com/suzuke/agend)。一個 daemon，管一整個 AI coding agent 艦隊，全部透過一個 Telegram bot 操作。

## AgEnD 不是什麼

不是又一個 coding agent。它自己不寫程式碼，不跟 Claude Code 或 Codex 或 Gemini CLI 競爭。

它是這些工具底下的 operations layer。

Claude Code 是開發者，AgEnD 是辦公室。桌子、通訊頻道、排程表、還有個人追蹤誰在做什麼。

## 實際遇到的問題

沒有 AgEnD 的時候，跑多個 AI agent 長這樣：

- 三個終端機分頁，各跑一個 Claude Code session
- 手機看不到任何一個
- 筆電休眠 session 就斷了
- Agent 完全不知道其他 agent 存在
- 沒有花費上限。我有過一次忘了管，一個 agent 一整晚燒了 $40
- Context window 默默填滿，回應品質默默下降。等你發現的時候 agent 已經在做蠢事了

## 架構

AgEnD 跑成系統服務（macOS 用 launchd，Linux 用 systemd）。每個 agent 在自己的 tmux 視窗裡透過 node-pty 生成，全部接到一個 Telegram bot。

```
┌─────────────────────────────────────────────┐
│                 Telegram Bot                 │
│          (一個 bot，多個 topic)               │
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

每個 Telegram Forum Topic 對應一個 agent instance。建 topic、綁專案目錄，agent 就起來了。刪 topic，agent 停。

## 五種 backend，一個介面

AgEnD 不管你用哪種 coding agent。目前支援：

| Backend | 認證方式 | 備註 |
|---------|---------|------|
| Claude Code | OAuth / API key | 主要支援，功能最完整 |
| OpenAI Codex | ChatGPT 登入 / API key | 支援 session resume |
| Gemini CLI | Google OAuth | |
| OpenCode | Provider 設定 | MCP instructions 尚未支援 |
| Kiro CLI | AWS Builder ID | 實驗性 |

Claude Code 跑後端、Codex 跑前端、Gemini 跑文件，全部在同一個 Telegram 聊天室管。哪個碰到 rate limit 就換一個，哪個模型適合就用哪個。

## 艦隊協調

這個功能別的地方沒有。

每個 agent 啟動時會拿到一個 MCP server。這個 server 提供艦隊工具：`list_instances`、`send_to_instance`、`delegate_task`、`request_information`、`report_result`。Agent 靠這些找到彼此、直接通訊。沒有中央調度器，點對點。

```
Agent A（後端）："前端，我改了 /users 的 API 回應格式"
Agent B（前端）："收到。正在更新 TypeScript 型別和 fetch 呼叫。"
```

靠的是 `send_to_instance`。每條訊息帶 metadata：`request_kind`（query/task/report/update）、`correlation_id` 串聯請求與回應、`task_summary` 讓 Telegram 看得到動態。

上面還有更高階的模式：

- `delegate_task(target, task, success_criteria)` -- 分派工作，定義什麼叫做完
- `request_information(target, question, context)` -- 問另一個 agent 事情
- `report_result(target, summary, artifacts, correlation_id)` -- 回報結果

General Topic 是自然語言調度器。訊息丟給它，它判斷哪個 agent 該接，或透過 `create_instance` 當場開一個新的。

## Context 自動輪替

跑久了 session 會退化。Context 填滿，模型忘記之前的指示，回應品質往下掉。用 Claude Code 跑一整天你一定遇過。

AgEnD 的 Context Guardian 盯著這件事。Context 超過 80% 或 session 超過設定時間（預設 8 小時），daemon 會：

1. 等 agent 閒下來
2. 擷取快照：最近的訊息、工具活動、statusline 資料
3. 結束 session，開一個新的
4. 快照注入為第一條訊息

快照大概 2000 tokens，200K context window 的 0.2%。Agent 帶著乾淨的 context 接著做。

之前試過讓 agent 自己寫交接報告再結束。不行。已經退化的 agent 沒辦法準確總結自己的狀態。現在整個流程由 daemon 從外部處理。

## 花費控管

放著不管的 agent 很燒錢。AgEnD 追蹤每個 instance 的花費，設每日上限。

```yaml
defaults:
  cost_guard:
    daily_limit_usd: 50
    warn_at_percentage: 80
    timezone: "Asia/Taipei"
```

80% 警告，100% 自動暫停，午夜重設。每天會發一份摘要到 General Topic，列出各 instance 花費、重啟次數、艦隊總計。

## 卡住偵測

Agent 有時候會卡住。跑進無限迴圈、等一個永遠不會來的輸入、或單純凍住。

卡住偵測器監控 15 分鐘無活動（可調）。觸發時，Telegram 跳通知加按鈕：「強制重啟」或「繼續等」。

## 模型容錯

Opus 碰到 rate limit，AgEnD 可以在下次 context 輪替時自動切到 Sonnet。

```yaml
instances:
  my-project:
    model: opus
    model_failover: ["sonnet"]
```

Telegram 會通知你。Rate limit 恢復後自動切回。

## 排程

SQLite 支撐的 cron 排程，daemon 重啟也不會消失。

```
你："每天早上九點，檢查有沒有需要 review 的 open PR"
Agent：→ create_schedule(cron: "0 9 * * *", message: "Check open PRs needing review")
```

排程有 rate-limit 意識。5 小時 API 用量超過 85%，觸發會延後，不浪費會被 throttle 的 token。

## 權限轉發

Claude Code 要跑危險操作（`rm`、`git push` 等）會彈批准。沒有 AgEnD 你得在終端機前面才能按。

AgEnD 把批准提示轉到 Telegram 變按鈕。手機就能批准或拒絕。常用的操作可以按「永遠允許」。

## 跟其他工具比

| | AgEnD | Overstory | Claude-to-IM | CrewAI / AutoGen |
|---|---|---|---|---|
| 定位 | Agent 艦隊 daemon | 多 agent 開發平台 | IM 橋接 Claude | 通用多 agent 框架 |
| Telegram 操控 | 原生支援 | 無 | 有 | 無 |
| 艦隊協調 | P2P via MCP tools | 內建 | 無 | 框架定義 |
| Backend 不限 | 5 種 backend | 自家 agent | 僅 Claude | LLM 不限但非 CLI agent |
| 持久化 | 系統服務 + SQLite | 雲端 | 不定 | 記憶體內 |
| 花費控管 | 每日上限 per instance | 平台管理 | 無 | 無 |
| Context 管理 | 自動輪替 | N/A | 無 | 無 |
| 目標使用者 | 跑 CLI agent 的開發者 | 採用 AI 開發的團隊 | 個人 Claude 使用者 | AI 應用開發者 |

CrewAI 和 AutoGen 是從零開始建多 agent 應用的框架。AgEnD 不用你重寫什麼，你把現在在用的 coding agent 丟給它就好。

## 開始用

```bash
# 前置需求
brew install tmux    # macOS

# 安裝
npm install -g @suzuke/agend

# 互動式設定
agend init

# 裝成系統服務並啟動
agend install --activate
```

`agend init` 引導你設定 Telegram bot、開第一個 instance、選 backend。設定檔在 `~/.agend/fleet.yaml`。

基本操作：

```bash
agend fleet start          # 啟動所有 instance
agend fleet stop           # 停止所有 instance
agend fleet status         # 顯示艦隊狀態
agend fleet logs my-proj   # 查看 instance 日誌
```

跑起來之後，打開 Telegram，在群組裡建一個 topic，就可以跟你的 agent 講話了。

## 接下來

AgEnD 目前 v1.11。核心流程（生成 agent、Telegram 管理、agent 之間協作）已經穩定。現在在做的：

- 更多 channel adapter。Discord 是 MVP，歡迎社群做 adapter。
- 跨 backend 相容性，尤其 OpenCode 和 Kiro 的 MCP instructions 支援。
- 團隊級別的協調模式。
- Webhook 整合外部監控。

Repo 在 [github.com/suzuke/agend](https://github.com/suzuke/agend)，MIT 授權。如果你已經在手動管多個 coding agent，可以省掉不少麻煩。
