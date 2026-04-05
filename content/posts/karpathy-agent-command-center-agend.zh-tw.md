---
title: "Karpathy 想要 Agent Command Center，我已經做了一個"
date: 2026-04-05
draft: false
summary: "Karpathy 跑了 8 個 agent 在 tmux 裡，說 'it's a mess'。他想要一個 agent command center IDE。AgEnD 從一月就在解決這些問題。"
description: "Karpathy 跑了 8 個 agent 在 tmux 裡，說 'it's a mess'。他想要一個 agent command center IDE。AgEnD 從一月就在解決這些問題。"
tags: ["agend", "karpathy", "ai-agent", "multi-agent", "agent-command-center", "developer-tools"]
ShowToc: true
TocOpen: true
---

Andrej Karpathy 最近在大量跑 agent。一次 8 個，4 個 Claude、4 個 Codex，每個配一張 GPU 跑 nanochat 實驗。他的評語：

> "it doesn't work and it's a mess"

幾天後他發了他真正想要的：

> "tmux grids are awesome, but I feel a need to have a proper 'agent command center' IDE for teams of them. I want to see/hide toggle them, see if any are idle, pop open related tools (e.g. terminal), stats (usage), etc."

還有更大的觀點：

> "the basic unit of interest is not one file but one agent. It's still programming."

我看到這些的時候想：我做這個做好幾個月了。不是 IDE 的形式，是 daemon 加 Telegram 介面。外觀不同，但解決的問題一模一樣。

以下是 Karpathy 描述的痛點，以及 [AgEnD](https://github.com/suzuke/agend) 怎麼處理每一個。

## 1. 盯著 tmux grid 看

Karpathy 的設定是一個 tmux grid 顯示所有 agent session。可以看到它們的終端機、看它們工作。問題是：你得坐在電腦前面。

他跑整夜的實驗。他睡覺，agent 繼續跑。但凌晨三點出了事，他早上才知道。

AgEnD 把整個操控面放到 Telegram。每個 agent 有自己的 Forum Topic，訊息即時進來。我在床上、咖啡廳、廁所都能查 agent 狀態。Agent 完成任務或出錯，手機就跳通知。

tmux session 還是存在的，AgEnD 透過 node-pty 在 tmux 裡跑每個 agent。但你不用盯著看。Telegram 就是你的窗口。

## 2. 用肉眼偵測 idle agent

> "I want to see if any are idle"

Karpathy 想要 idle 偵測。現在他靠肉眼掃 tmux pane，看哪些 agent 沒在輸出了。

AgEnD 的 Hang Detector 監控無活動狀態，預設 15 分鐘，可調。觸發時 Telegram 跳通知加兩個按鈕：「強制重啟」或「繼續等」。Topic icon 也會更新，一眼就能看出哪些在跑、哪些停了。

你不用一直看，它替你看。

## 3. Watcher scripts 是自己寫的膠水

Karpathy 提到「building watcher scripts to keep them looping」。這些是他自己的 script，沒開源，專門監控 agent 然後在它們停下來時重啟。

AgEnD 把這些全部內建了：

- Crash recovery，可設 restart policy（最大重試次數、exponential backoff）
- 每個 backend 的 error pattern 偵測（rate limit、auth 失敗、usage limit）
- Model failover：Opus 碰到 rate limit，自動切 Sonnet
- Context rotation：context 超過 80%，daemon 擷取快照、砍 session、開新的、注入快照

你不寫 watcher script，你寫 YAML 設定檔。

## 4. Agent 間協調 "a mess"

Karpathy 試了 8 個 agent 協作。他的協調方式：每個研究方向一個 git branch，git worktree 隔離，用檔案互傳訊息。他說 a mess。

AgEnD 的艦隊協調用啟動時注入的 MCP tools：

- `send_to_instance` -- 點對點訊息，帶結構化 metadata
- `delegate_task` -- 分派工作，定義成功標準
- `request_information` -- 問另一個 agent 事情
- `report_result` -- 回報結果，帶 correlation ID

沒有中央調度。每個 agent 透過 `list_instances` 發現其他人，直接通訊。General Topic 自動把自然語言請求路由到對的 agent。

不是說 agent 就不會做蠢事了。但至少通訊管道不是「檔案」。

## 5. 沒有統一的 usage/cost 追蹤

> "stats (usage)"

Karpathy 想要 usage 統計。8 個 agent 同時跑，花費加很快，但沒有一個地方能看到總數。

AgEnD 的 Cost Guard 追蹤每個 instance 的花費，設每日上限：

```yaml
defaults:
  cost_guard:
    daily_limit_usd: 50
    warn_at_percentage: 80
```

80% 警告，100% 自動暫停。每天發一份摘要到 General Topic，列出各 instance 花費、重啟次數、艦隊總計。睡醒就知道每個 agent 燒了多少。

## 6. 成果散落在 git branches

Karpathy 的 autoresearch 每個實驗一個 branch。成果散在各 branch 裡，得手動檢查哪些有進展。

AgEnD 不取代 git branching，那還是 agent 隔離工作的方式。但 `report_result` 給了結果回流一個結構。Agent B 完成 Agent A 委派的任務時，會發一份結構化報告，帶 summary、artifacts、correlation_id 串回原始請求。這些出現在 Telegram 裡，有上下文。

Shared Decisions（`post_decision` 和 `list_decisions`）讓 agent 記錄跨艦隊的架構決策。不是 git history，更像是每個 agent 都能讀的 ADR log。

## 7. 多 backend 切換麻煩

Karpathy 同時跑 Claude 和 Codex。不同 model 做不同事。但切換意味著不同 CLI、不同認證、不同的雷。

AgEnD 把這層抽象掉了。五種 backend 一個介面：

- Claude Code（OAuth / API key）
- OpenAI Codex（ChatGPT 登入 / API key）
- Gemini CLI（Google OAuth）
- OpenCode（provider 設定）
- Kiro CLI（AWS Builder ID）

每個 instance 在 `fleet.yaml` 裡選 backend。Claude 跑後端、Codex 跑前端、Gemini 跑文件。碰到 rate limit 就換。Telegram 介面完全一樣。

## 8. 知識不持久

聊天記錄在 session 結束時就消失。Context 填滿被輪替。Agent 忘了它學過的東西。

AgEnD 的 Context Guardian 在輪替前擷取快照：最近的訊息、工具活動、statusline 資料，大概 2000 tokens。新 session 啟動時注入這份快照。不完美，但 agent 不會從零開始。

Shared Decisions 也有幫助。Agent 做了架構決策，可以透過 `post_decision` 記錄。其他 agent（以及同一個 agent 的未來 session）透過 `list_decisions` 讀取。知識比單一 session 活得久。

## 外觀不同，問題一樣

Karpathy 想要 IDE。有面板、有開關、有統計的視覺化 command center。AgEnD 是 daemon 加 Telegram bot。形狀不同，底層架構一樣：

| Karpathy 想要的 | AgEnD 的對應 |
|---|---|
| 視覺化 agent 面板 | Telegram Forum Topics |
| Idle 偵測 | Hang Detector + topic icon |
| Watcher scripts | 內建 crash recovery + failover |
| Agent 協調 | MCP fleet tools（P2P） |
| Usage 統計 | Cost Guard + Daily Summary |
| 成果追蹤 | report_result + Shared Decisions |
| 多 backend | 5 種 backend 抽象 |
| 知識持久化 | Context rotation snapshot + Shared Decisions |

形式的問題是真的。有人想要桌面 app，有面板可以拖來拖去。我想要凌晨兩點能從手機用的東西。兩種都合理。

但底層問題是一樣的，而且是在 daemon 層解決的，不是 UI 層。AgEnD 明天可以加一個桌面前端，核心不用動。Fleet manager、cost guard、hang detector、context rotation，全部是 backend 邏輯。Telegram 只是一個 adapter。

## Karpathy 說對了什麼

Karpathy 說的最有用的不是關於工具：

> "the basic unit of interest is not one file but one agent. It's still programming."

完全正確。我們沒有超越 IDE 的時代。我們需要新的 IDE，給新的工作單位用。舊 IDE 幫你管檔案，新 IDE 幫你管 agent。

AgEnD 是一個答案。會有其他的。但如果你現在還在用 tmux grid 和 watcher script 管 agent，不用等完美的 IDE。問題現在就能解決。

Repo 在 [github.com/suzuke/agend](https://github.com/suzuke/agend)，MIT 授權。
