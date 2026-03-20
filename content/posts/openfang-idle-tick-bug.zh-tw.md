---
title: "我的 AI Agent 整晚空轉燒 token"
date: 2026-03-20
draft: false
summary: "OpenFang 跑了一整晚，170 次 LLM 呼叫，其中 80% 的回覆是「沒事做」。以下是我在排程程式碼裡挖出的 bug。"
description: "OpenFang 跑了一整晚，170 次 LLM 呼叫，其中 80% 的回覆是「沒事做」。以下是我在排程程式碼裡挖出的 bug。"
tags: ["ai", "agent", "openfang", "debugging", "rust"]
ShowToc: true
TocOpen: true
---

這週看 Claude Code 用量的時候覺得哪裡不對。某天跑了 809 個 session，平常大概 100 上下。

我在筆電上把 [OpenFang](https://github.com/RightNow-AI/openfang) 當 daemon 跑。幾天前在 dashboard 上啟用了內建的 Researcher Hand，用了一次，然後就忘了。

它沒忘。

## 數字

翻了 `~/.claude/projects/` 底下的 session log，openfang 專案有 497 個 session 檔，幾乎全是同一天產生的。

把 JSONL 解析完，畫面很清楚：

| 類別 | Sessions | 資料量 |
|---|---|---|
| Code Reviewer（superpowers 插件） | 346 | 22.9M |
| Researcher Hand（openfang） | 170 | 19.7M |
| Conversation Summarizer | 21 | 0.3M |
| 其他 | 7 | 2.8M |

346 個 code reviewer session 是 superpowers 插件在開發過程中做 review，正常。

170 個 Researcher Hand session 不正常。

## 每 60 秒，精準如鐘錶

看 timestamp 就知道了：

```
2026-03-19T13:06:57
2026-03-19T13:07:57
2026-03-19T13:08:57
2026-03-19T13:09:57
2026-03-19T13:10:57
...
```

一分鐘一個 session。連續 12 小時。

每個 session 都以同一段訊息開頭：

```
[AUTONOMOUS TICK] You are running in continuous mode.
Check your goals, review shared memory for pending tasks,
and take any necessary actions.
```

然後 80% 的時間，回覆是：

```
NO_REPLY
```

Agent 醒來，東看西看，發現沒事做，繼續睡。但「睡」還是得做一次完整的 LLM round-trip：system prompt、tool 定義、整個 context window 都送出去。每 60 秒一次。

最終統計：
- 169/170 個 session 包含 `[AUTONOMOUS TICK]` 訊息
- 136/170（80%）回覆 `NO_REPLY`
- 35/170（20%）有做正事
- 有一個 session 的結尾是「You've hit your limit -- resets 3am」

## 找 bug

OpenFang 有個概念叫「Hands」——打包好的 agent 擴充套件，自帶 system prompt、工具和設定。Researcher Hand 就是其中之一。它的設定檔寫了 `max_iterations = 80`，本意是限制 agent 單次對話最多走幾步。

但 `kernel.rs` 裡有這麼一段：

```rust
schedule: if def.agent.max_iterations.is_some() {
    ScheduleMode::Continuous {
        check_interval_secs: 60,
    }
} else {
    ScheduleMode::default() // Reactive
},
```

只要 `max_iterations` 有值，Hand 就會被塞進 Continuous 模式，60 秒 tick 一次。程式碼把「這個 agent 可以走多步」當成了「這個 agent 要自主循環運行」。完全是兩回事。

30 個內建 Hand 裡有 8 個設了 `max_iterations`：browser、clip、collector、lead、predictor、researcher、trader、twitter。只要啟用任何一個，不管有沒有工作都會每分鐘 tick 一次。

更慘的是：啟用過的 Hand 會被寫進 `~/.openfang/hand_state.json`。每次 daemon 重啟都會自動恢復。我只是在 dashboard 上點了一下 Researcher Hand，之後忘了關，它就一直 tick 到現在。

## 修法不難

`max_iterations` 和排程模式需要拆開。可以在 HAND.toml 裡加一個明確的 `autonomous = true` 欄位，讓真正需要自主跑的 Hand 自己聲明。

空轉的 tick 也應該退避。如果 agent 連續三次回 `NO_REPLY`，間隔從 60 秒拉到 5 分鐘，再拉到 15 分鐘、30 分鐘。明明什麼事都沒有，每分鐘問一次有什麼意義。

已經開了 [issue #756](https://github.com/RightNow-AI/openfang/issues/756)。

## 教訓

如果你也在跑 OpenFang，去看一下 `~/.openfang/hand_state.json`。裡面有沒有你不在用的 Hand？有的話刪掉，重啟 daemon。不然那些 agent 會每分鐘醒來一次，對著虛空發 LLM 呼叫。

我弄了一個 research agent 幫我查資料，結果它教我最深刻的一課是——養一個閒著的 agent 有多燒錢。
