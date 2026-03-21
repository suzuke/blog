---
title: "讓 Claude Code 永遠在線：我做了一個 Daemon 來管它"
date: 2026-03-21
draft: false
summary: "Claude Code 的 Telegram plugin 要求終端機一直開著。我寫了一個 daemon 來解決這件事，順便處理了 context 爆滿、遠端權限批准、語音轉錄等問題。"
description: "Claude Code 的 Telegram plugin 要求終端機一直開著。我寫了一個 daemon 來解決這件事，順便處理了 context 爆滿、遠端權限批准、語音轉錄等問題。"
tags: ["claude-code", "telegram", "daemon", "ai-agent", "node-pty"]
ShowToc: true
TocOpen: true
---

昨天寫了一篇 Telegram 遙控 Claude Code 的設定指南。設定完很開心，躺沙發上手機就能叫 agent 改 code、跑測試、查資料。

用沒多久，問題來了。

## 這東西不能關

Telegram plugin 就是一個 MCP server，掛在 Claude Code CLI 的 session 上面。session 在，bot 就在。session 掛了，bot 就斷了。

白話講：你的終端機要永遠開著。

關上筆電蓋子？斷了。SSH 逾時？斷了。手滑按到 Ctrl+C？斷了。macOS 更新重開機？也斷了。

太脆弱了。我要的是像 PostgreSQL 那種東西——開機自動跑，掛了自動起來，人不在電腦前也照常。

## Context 會滿

就算終端機一直開著，還有 context window 的問題。

同一個 session 聊越久，context 用越多。叫它讀了幾十個檔案、跑了一堆指令，context 就快爆了。爆了之後 Claude 的表現明顯變差——回應變慢、忘記前面的東西、判斷力下降。

我看過一個研究，context 維持在 40% 以下的時候 agent 表現最好。超過就開始走下坡。

所以除了讓 session 活著，還要在 context 快滿的時候自動換新 session。

## 遠端批准

第三個問題：權限。

Claude Code 跑 `rm`、`git push` 這種危險操作會在終端機跳批准提示。但我人在外面，手機上只有 Telegram，看不到終端機。

全部自動允許？太危險。agent 判斷失誤一次，`rm -rf` 下去專案就沒了。

得有辦法把批准提示轉到 Telegram，手機上按個按鈕決定要不要放行。

## 做了什麼

花了幾個小時寫了 [claude-channel-daemon](https://github.com/suzuke/claude-channel-daemon)。Node.js 程式，透過 `node-pty` 開一個 pseudo-terminal 跑 Claude Code CLI。

分四塊：

**Process Manager** — 啟動 Claude Code、管 session resume、掛了自動重啟。重啟用指數退避（1 秒、2 秒、4 秒、8 秒……最多 60 秒），穩定跑超過五分鐘就歸零。

**Context Guardian** — 每兩秒讀 Claude 的 status line JSON 看 context 使用量。超過閾值就殺掉舊 session、清 session ID、開新的。閾值我設 40%。

**Memory Layer** — 用 chokidar 盯著 Claude 的記憶目錄，有變動就備份到 SQLite。session 換了，記憶不丟。

**Service Installer** — 吐 macOS launchd 或 Linux systemd 的服務檔，開機自動啟動。

## 批准系統放哪裡

這邊走了一段彎路。

一開始想在 daemon 裡面開 HTTP server 來處理批准。寫完才發現不對——Telegram plugin（grammy）已經在 long-poll `getUpdates` 了，daemon 再開一個 poller 就會 409 衝突。Telegram 只允許一個 consumer。

最後決定做在 plugin 裡面。plugin 已經有 bot 實例跟 `callback_query` handler，直接用就好。

做法：在 plugin 裡面開一個 `Bun.serve` HTTP server，聽 `127.0.0.1:18321`。Claude Code 的 PreToolUse hook 每次工具調用都會 POST 到這裡。

收到之後用 regex 判斷危不危險——`rm`、`sudo`、`git push --force`、還有 `.env` 和 `.claude/settings.json` 這些敏感路徑。安全的直接放行，危險的發 Telegram 訊息給我，帶兩個按鈕：✅ Approve 跟 ❌ Deny。

按了就回應 hook，Claude 繼續或停下來。兩分鐘沒按就自動拒絕。

## 語音轉錄

官方 plugin 收到語音只傳 file_id，不轉文字。但我大部分時間都用語音輸入，所以加了 Groq 的 Whisper API（`whisper-large-v3-turbo`）。

流程很直接：收到語音 → 下載 OGG → 打 Groq API → 拿到文字 → 加 `[voice message transcription]` 前綴送進 Claude。中文辨識率可以，偶爾同音字會錯，但意思通常抓得到。

## 維護改過的 Plugin

改了官方 plugin 就要想：官方更新的時候怎麼辦？

我 fork 了 [claude-plugins-official](https://github.com/anthropics/claude-plugins-official)，把語音轉錄跟批准系統拆成兩個獨立 feature branch。然後加了 GitHub Actions，每天自動跑：

1. fetch upstream 最新版
2. merge 到我的 main
3. 分別 merge 到兩個 feature branch
4. 合併成一個 deploy branch
5. 有衝突就開 issue 通知我

部署就一行 `cp`，把 deploy branch 的 `server.ts` 複製到 `~/.claude/plugins/cache/` 就好。

## 跑起來怎麼樣

剛跑起來，目前看起來還行。session 輪替蠻順的，換完之後 Claude 的記憶系統自動載入，還是記得之前聊過什麼。遠端批准也很方便，手機收到通知按一下就好。

context 閾值設 40% 的效果要再觀察一陣子，但至少目前 agent 的回應品質跟速度都不錯。

程式碼都在 [GitHub](https://github.com/suzuke/claude-channel-daemon) 上，MIT 授權。如果你也在用 Claude Code 的 Telegram channel，可以參考看看。
