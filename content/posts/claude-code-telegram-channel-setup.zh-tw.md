---
title: "用 Telegram 遙控 Claude Code：五分鐘設定指南"
date: 2026-03-20
draft: false
summary: "Claude Code 支援 Telegram Channel 了。設定一個 bot，配對完就能從手機操控你的 agent。"
description: "Claude Code 支援 Telegram Channel 了。設定一個 bot，配對完就能從手機操控你的 agent。"
tags: ["claude-code", "telegram", "ai-agent", "setup-guide"]
ShowToc: true
TocOpen: true
---

我大部分時間都在終端機裡跑 Claude Code。但有時候人不在電腦前，躺沙發上或出門在外，突然想叫 agent 做點事。以前我用手機 SSH 進去，能用是能用，但在那個小螢幕上打指令真的很痛苦。

我一直想解決這個問題，甚至已經開始自己寫 MCP server 來串 Telegram。研究了一堆 Claude 衍生產品，結果發現 Claude Code 官方直接做好了，而且做法跟我的想法很接近——都是靠 MCP server 當中間層來跟 Telegram 串接。省了我自己造輪子的時間。

把一個 Telegram bot 接上你正在跑的 session，傳訊息給 bot 就是在跟 agent 對話，回覆直接出現在聊天裡。白話講就是用 Telegram 傳訊息給你的終端機。

設定大概五分鐘。

## 你需要什麼

- Claude Code v2.1.80 以上
- [Bun](https://bun.sh) runtime（plugin server 要用）
- 一個 Telegram 帳號

## 第一步：建立 Telegram bot

打開 Telegram，找 [@BotFather](https://t.me/BotFather)，送出 `/newbot`。它會問你顯示名稱跟 username（結尾要是 `bot`）。完成後會給你一串 token，像 `123456789:AAHfiqksKZ8...`，整串複製起來。

## 第二步：裝 plugin

在 Claude Code 裡：

```
/plugin install telegram@claude-plugins-official
```

好了。

## 第三步：設定 bot token

```
/telegram:configure 你的TOKEN貼在這
```

Token 會寫進 `~/.claude/channels/telegram/.env`。不想在 session 裡貼 token 的話，直接用編輯器改那個檔案也行。

## 第四步：重新啟動

退出目前的 session，加上 `--channels` flag 重開：

```bash
claude --channels plugin:telegram@claude-plugins-official
```

Telegram MCP server 會跟著啟動，連上 Bot API。plugin list 裡應該看得到。

## 第五步：配對

打開 Telegram，私訊你的 bot。它會回一組六碼的配對碼，像 `a4f91c`。回到 Claude Code：

```
/telegram:access pair a4f91c
```

你的 Telegram user ID 就進白名單了。從這時候開始，你傳給 bot 的訊息都會送到 Claude。

## 第六步：鎖起來

bot 預設是 `pairing` 模式，隨便一個人私訊它都會拿到配對碼。不要讓它一直開著。配對完所有需要存取的人之後：

```
/telegram:access policy allowlist
```

陌生人什麼都收不到，沒有錯誤訊息、沒有配對碼，就是已讀不回。

## 用起來是什麼感覺

我在手機上傳「看一下 deploy 成功了沒」給 bot。Claude 跑 `gh run list`，在 Telegram 裡回我結果。照片也能傳，bot 會下載下來給 Claude 看。

bot 能回訊息（文字加附件都行）、加 emoji 反應（我拿來當「收到」的信號）、還能改之前送出的訊息（「處理中…」變成實際結果）。

## 幾件事要知道

MCP server 跑在你自己電腦上，除了 Telegram Bot API 收發訊息之外，資料不經過第三方。

Bot API 沒有聊天記錄功能，bot 只看得到即時送來的訊息。要引用之前的東西，自己貼。

改 token 要重啟（或跑 `/reload-plugins`）。存取控制不用，`access.json` 每則訊息進來都會重讀，改了馬上生效。

## 存取管理速查

```bash
# 看目前狀態
/telegram:access

# 直接用 Telegram 數字 ID 加人
/telegram:access allow 412587349

# 移除
/telegram:access remove 412587349

# 核准配對
/telegram:access pair <code>

# 切 policy
/telegram:access policy pairing    # 暫時的，讓新使用者配對用
/telegram:access policy allowlist  # 長期的，鎖定已知使用者
/telegram:access policy disabled   # 緊急開關，全部拒絕
```

要查某人的 Telegram 數字 ID，叫他們傳訊息給 [@userinfobot](https://t.me/userinfobot)。

## 為什麼不用 SSH 就好

我用手機 SSH 撐了好幾個月。螢幕太小看了眼睛痛，tmux 和 mosh 多一層要管，複製貼上 output 更是災難。

Telegram 有正常的聊天介面，圖片跟 code block 都能看，client 端還保留聊天記錄。手機上不用再開一個 SSH terminal，用 Telegram 聊天就好。電腦上的 Claude Code session 還是要開著，bot 才會動。但因為 Telegram 訊息進的是同一個 session，你裝過的 skills、設定過的記憶、所有 plugin 都在，跟坐在電腦前用完全一樣。
