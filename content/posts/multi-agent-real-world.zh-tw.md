---
title: "Multi-Agent 已經在跑了：一個 AI 開發團隊的實戰紀錄"
date: 2026-03-28
draft: false
summary: "學術論文說 multi-agent 在平行任務上提升 80%，在序列任務上反而掉 70%。我們用 CCD 跑了一天真實開發，數字跟論文說的差不多 — 但魔鬼在細節裡。"
description: "學術論文說 multi-agent 在平行任務上提升 80%，在序列任務上反而掉 70%。我們用 CCD 跑了一天真實開發，數字跟論文說的差不多 — 但魔鬼在細節裡。"
tags: ["multi-agent", "ai-agent", "claude-code", "ccd", "developer-tools"]
ShowToc: true
TocOpen: true
---

今天做了一個實驗。

手上有一個中型專案要做 code review、修 bug、開發新 feature，我決定全部交給 AI agent 做，而且不只用一個。三個 agent 同時跑，各自負責不同的事，透過 [claude-channel-daemon](https://github.com/suzuke/claude-channel-daemon)（CCD）的 P2P 通訊互相協調。

跑了一整天下來，有些結果跟預期一樣，有些完全出乎意料。

## 學術界怎麼說

先看看理論。Google Research 今年初發了一篇 [Towards a Science of Scaling Agent Systems](https://arxiv.org/abs/2512.08296)，用 180 種 agent 配置做了大規模實驗。結論很清楚：

multi-agent 在可平行化的任務上提升 80.8%，但在序列任務上掉 39% 到 70%。

這篇論文還發現三個效應：工具越多協調越貴（agent 要用很多工具的任務，光是協調誰用什麼就拖慢速度）、能力飽和（一個 agent 已經做得夠好的任務，加人沒什麼幫助）、錯誤放大（去中心化架構下，一個 agent 的錯會擴散到其他 agent）。

台大李宏毅教授在 AI Agent 課程裡也做了拓撲結構的比較 — Chain、Star、Tree、Mesh、Random。結論是 Mesh（所有 agent 互相連通）通常效果最好，而且 agent 越多通常越好，但前提是任務可以拆。

SWE-bench 的數據也有佐證：multi-agent（manager + researcher + engineer + reviewer）比 single-agent 高了 7.2%。客服路由任務更誇張，multi-agent 準確率 97-99%，single-agent 只有 79-86%。

聽起來很美。但實際用起來呢？

## 三個 Agent 做 Code Review

第一個實驗是 code review。把一個 PR 同時丟給三個 agent 看：一個負責架構層面、一個看程式碼品質、一個檢查 UX 和使用者流程。

三個 agent 各自分析完後，透過 CCD 的 `send_to_instance` 把結果彙總到 General instance。General 扮演 orchestrator，負責去重、排優先順序、篩掉誤報。

結果：三個 agent 合起來提了 26 個問題。General 交叉比對後篩到 11 個真實問題，其中 8 個是誤報（一個 agent 覺得有問題但其他兩個認為沒事）。最後留下的 11 個問題品質很高，我看完覺得每個都該修。

這跟 Grok 4.20 的辯論架構邏輯類似。Grok 4.20 用四個 persona（隊長、研究員、邏輯官、反面人）平行分析再互相辯論，幻覺率降了 65%。我們的做法沒那麼精緻 — 沒有顯式的辯論機制，只是靠交叉驗證 — 但效果的方向一致：多個視角看同一件事，錯誤更容易被抓出來。

## 平行開發 Feature

第二個實驗是同時開發兩個獨立的 feature。兩個 agent 各自在獨立的 git worktree 裡工作，完全不互相干擾。做完後 General instance 負責 review 和 merge。

這個場景效果最好。兩個 feature 本來就沒有依賴關係，平行跑就是快兩倍。Google 那篇論文說平行任務 +80.8%，我們的體感也差不多。

但有一個坑：兩個 agent 各自改了相關的檔案，merge 的時候出了衝突。這時候 General 要判斷怎麼解 — 它看了兩邊的改動，理解各自的意圖，選了比較合理的那個。大部分時候判斷是對的，但偶爾會出錯，需要人介入。

## 多 Agent 搜尋研究

第三個實驗是研究型任務。要調查「AI agent 自動賺錢的真實案例」，三個 agent 各自搜尋不同方向 — 一個查 freelance 套利、一個查內容自動化、一個查 crypto 和電商。

結果彙整起來涵蓋面很廣，每個 agent 找到的東西幾乎不重複。這是 multi-agent 最直覺的好處：平行搜尋本來就比一個人輪流搜快。

## 什麼時候不該用 Multi-Agent

跑了一天下來，最大的收穫不是「multi-agent 很厲害」，而是「知道什麼時候不該用」。

序列任務用 multi-agent 反而變慢。有一個 bug 需要先讀 log、再追 call stack、再看相關程式碼、再想修法 — 這種前後依賴的工作，一個 agent 從頭做到尾最快。拆給多個 agent 只會增加溝通開銷。

Google 那篇論文的數據很精準：序列任務掉 39-70%。我的體感是，只要任務有明確的前後步驟，就不要拆。

另一個陷阱是 agent 之間的衝突。兩個 agent 同時改同一個檔案，最後要人來解。如果任務的邊界切不乾淨，multi-agent 的協調成本會吃掉平行帶來的好處。

## 不需要特殊框架

一個有趣的發現：我們整天用的 multi-agent 協作，沒有用任何 multi-agent 框架。沒有 LangGraph、沒有 CrewAI、沒有 AutoGen。

CCD 提供了三個原語：fleet（多個 Claude Code instance）、`send_to_instance`（互相傳訊）、worktree（獨立工作空間）。就這三樣，夠了。

General instance 自己學會了怎麼當 orchestrator — 拆解任務、分配給其他 instance、收回結果、review、merge、回報。這些行為都是 Claude Code 自己推導出來的，沒有人寫 orchestration 邏輯。

Grok 4.20 用了精心設計的四個 persona 和辯論流程。我們什麼都沒設計，只是給了 agent 互相溝通的工具，它們自己長出了協作模式。效果沒有 Grok 那麼精緻，但對日常開發來說夠用了。

## 數字總結

| 場景 | 方法 | 效果 |
|---|---|---|
| Code Review | 3 agent 平行 + 交叉驗證 | 26 問題 → 11 真實問題，過濾率 58% |
| Feature 開發 | 2 agent 平行 worktree | 接近 2x 速度，但需要處理 merge 衝突 |
| 研究搜尋 | 3 agent 分方向搜尋 | 涵蓋面廣，幾乎零重複 |
| Bug 修復 | Single agent | 最快，multi-agent 反而慢 |

跟學術數據的對照：Google 說平行任務 +80.8%，我們的 feature 開發體感一致。Google 說序列任務 -39~70%，我們的 bug 修復也證實了這點。SWE-bench 說 multi-agent +7.2%，我們的 code review 交叉驗證確實提升了品質。

## 回到本質

Multi-agent 的價值在一句話：讓多個不同視角同時看同一件事。

學術上叫 diversity of thought、ensemble method、cross-validation。實務上就是：一個 agent 會自信地犯錯，但三個 agent 看同一段 code，錯誤更容易被抓出來。

關鍵是知道什麼時候該用。可以拆的任務，越多 agent 越好。不能拆的任務，一個就夠。拆錯了比不拆更慢。

工具方面，不需要複雜的框架。只要 agent 之間能互相傳訊、有獨立的工作空間，協作模式會自然出現。CCD 就是在做這件事 — 提供基礎設施，讓 agent 自己長出團隊合作的能力。

---

相關連結：
- [Google: Towards a Science of Scaling Agent Systems](https://arxiv.org/abs/2512.08296)
- [Grok 4.20 Multi-Agent 架構](https://www.buildfastwithai.com/blogs/grok-4-20-beta-explained-2026)
- [SWE-bench Multi-Agent 結果](https://dev.to/nikita_benkovich_eb86e54d/coding-agent-teams-outperform-solo-agents-722-on-swe-bench-verified-4of5)
- [claude-channel-daemon (CCD)](https://github.com/suzuke/claude-channel-daemon)
