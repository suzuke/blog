---
title: "Multi-agent 協作的下一步：跨體系混合才能克服技術偏見"
date: 2026-03-30
draft: false
summary: "CCD 和 Scion 在架構上幾乎一樣，但 Scion 已經做到不同 LLM 體系混合跑。這不只是功能差異——不同模型有不同的訓練偏見，混合才能互相糾偏。"
description: "CCD 和 Scion 在架構上幾乎一樣，但 Scion 已經做到不同 LLM 體系混合跑。這不只是功能差異——不同模型有不同的訓練偏見，混合才能互相糾偏。"
tags: ["multi-agent", "agent", "claude-code", "open-source", "developer-tools", "gemini"]
ShowToc: true
TocOpen: true
---

CCD 和 [Scion](https://github.com/GoogleCloudPlatform/scion) 解決的問題幾乎一樣：多個 AI 代理並行跑、互相協調、完成一個代理做不完的工作。底層架構也很像，都用 tmux 管理 sessions，都讓每個代理有獨立的工作空間。

但有一個地方差很多，而且這個差異指向了 multi-agent 協作真正重要的東西。

## 架構上的相似

先說共同點，因為這值得強調。

兩個框架都是同一類工具：

- 多個 AI 代理同時跑
- 每個代理有獨立的 git worktree
- 代理之間可以互相協調
- 都用 tmux 管理 sessions

Scion 用容器（Docker/Podman/K8s）隔離每個代理，CCD 用進程隔離。這是實作上的差異，但解決的問題是一樣的：怎麼讓多個代理不互相干擾地同時工作。

## 真正的差異：誰跑在裡面

CCD 目前只支援 Claude Code 作為底層引擎。每個 instance 都是一個 Claude Code，沒有別的選擇。你有十個代理，就是十個 Claude。

Scion 是 **harness-agnostic**。它可以同時跑 Gemini CLI、Claude Code、Codex、OpenCode，或任何可以在容器裡跑的 LLM CLI 工具。每個代理容器有自己的 home directory、credentials、config，完全獨立。你可以安排一個 Claude 代理和一個 Gemini 代理同時看同一段程式碼。

這不只是「支援更多工具」。這是完全不同的一件事。

## 為什麼跨體系協作重要

每個 LLM 都有它的訓練偏見。

Claude 有 Claude 的思維習慣——它的訓練數據、RLHF 方向、架構設計，都讓它在某些問題上有特定的傾向。Gemini 有 Gemini 的。Codex 有 Codex 的。這不是哪個比較好的問題，是每個模型的訓練歷史決定了它「習慣用什麼方式看問題」。

單純用多個 Claude 代理並行，你得到的是多個帶著同樣偏見的視角。它們可能同時犯同一種錯，而且互相強化。三個 Claude 一起做 code review，如果 Claude 在某類問題上有系統性的盲點，三個都會漏掉。

跨體系協作打破了這個問題。Claude 看不到的，Gemini 可能看到；Gemini 習慣的解法，Claude 可能會質疑。不同訓練背景帶來真正不同的視角，而不只是「更多算力」。

這才是 multi-agent 真正的價值——不是平行化，是多樣性。

## Scion 的做法

Scion 的設計哲學叫 "less is more"：不規定固定的 orchestration 模式，給每個代理一組工具，讓模型自己決定如何協調。

因為是 harness-agnostic，一個 Gemini 代理和一個 Claude 代理可以用同一套工具互相溝通，各自用自己的語言模型推理，但在同一個任務框架下工作。協調的邏輯由模型自己推導，不是寫死的。

這在實驗層面很有意思：不同體系的代理在沒有人規定「你要怎麼合作」的情況下，會長出什麼樣的協作模式？

## CCD 的現況和下一步

CCD 現在是 Claude Code 的編排層。它做得很好的事是：從手機管多個 Claude Code instance、持久化排程、context 自動輪換、P2P 跨 instance 通訊。

但寄生在單一 LLM 體系下，有一個根本限制：所有代理帶著同樣的訓練偏見。

往 harness-agnostic 走是很自然的下一步。用相同的 tmux-based 架構，但讓每個 instance 可以是不同的 CLI Agent——不只是 Claude Code，也可以是 Gemini CLI、Codex 或其他工具。界面（Telegram bot）保持不變，排程和協作機制也保持不變，只是代理的來源變多樣。

這樣你就能在手機上指揮一個真正多樣的 AI 開發團隊：一個 Claude 負責架構設計，一個 Gemini 做安全審查，一個 Codex 專注程式碼最佳化。它們帶著不同的訓練背景看同一個問題，互相糾偏。

---

Multi-agent 的第一個版本是「多一些代理」。下一個版本是「多樣化的代理」。Scion 已經在往那邊走了。

---

相關連結：
- [claude-channel-daemon (CCD)](https://github.com/suzuke/claude-channel-daemon)
- [Scion (Google Cloud Platform)](https://github.com/GoogleCloudPlatform/scion)
- [Scion agents.md](https://github.com/GoogleCloudPlatform/scion/blob/main/agents.md)
