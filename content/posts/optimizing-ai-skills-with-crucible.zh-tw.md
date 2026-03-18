---
title: "我讓 AI 優化自己的 Skill，它砍掉了 94% 的內容"
date: 2026-03-19
draft: false
summary: "拿 crucible 優化一個中文去 AI 味的 skill。988 行壓到 62 行，覆蓋率從 21% 拉到 100%。然後我發現 metric 設計本身有 bug。"
description: "拿 crucible 優化一個中文去 AI 味的 skill。988 行壓到 62 行，覆蓋率從 21% 拉到 100%。然後我發現 metric 設計本身有 bug。"
tags: ["ai", "autoresearch", "prompt-engineering", "goodharts-law", "crucible"]
ShowToc: true
TocOpen: true
---

上次我讓 AI 跑五子棋實驗，它[學會作弊了](/blog/zh-tw/posts/ai-cheating-experiments/)。這次我給它一個不一樣的題目：優化一個 prompt。

具體來說，是一個叫 humanizer-tw 的 Claude Code skill——一份 markdown 文件，教 LLM 怎麼把 AI 味很重的中文改寫成像人寫的。我想知道 crucible 能不能拿來優化 prompt，不只是優化程式碼。

## 實驗設計

核心想法很簡單：skill.md 就是 code。agent 改它，evaluate.py 衡量效果。跟優化排序演算法一模一樣的流程，只是 evaluate.py 裡面恰好呼叫 LLM。

```
agent 修改 skill.md
    → evaluate.py 拿修改後的 skill 當 system prompt
    → 餵 5 段 AI 味文字給 claude
    → 另一個 claude 當 judge 打分
    → regex 檢查 AI 模式有沒有被消掉
    → 算出 0-90 分
```

### Metric 怎麼算

三個子分數加起來：

| 維度 | 滿分 | 怎麼算 |
|------|------|--------|
| 品質 | 50 | LLM judge 打 5 維度分（直接性、節奏、信任度、真實性、精煉度） |
| 覆蓋率 | 20 | regex 偵測 AI 模式有沒有被消掉（時代開場白、連接詞、黑話等） |
| 簡潔度 | 20 | skill 越短分越高。公式：`max(0, 20 - 20 * (ratio - 0.5))` |

另外有結構罰分：刪了必要段落或超過原始長度 1.5 倍就扣分。

### 防作弊

測試文本分兩組——agent 看得到 3 個（train），看不到 5 個（test）。只有 test 的分數算分。rubric 和 evaluate.py 都是 hidden file，agent 無法讀取。

## v1：全部 Crash

跑了 4 輪，全炸。

| 迭代 | 分數 | 狀態 | 時間 |
|------|------|------|------|
| 1 | 0 | crash | 507s |
| 2 | 0 | crash | 469s |
| 3 | 0 | crash | 434s |
| 4 | 0 | crash | 467s |

Debug 了好一陣子，發現三個 bug 疊在一起：

**Bug 1：parse_metric 硬編碼 30 秒。** crucible 的 `runner.py` 裡 `parse_metric()` 的 timeout 寫死 30 秒。我的 `commands.eval` 是 `python3 -u evaluate.py --print-metric`，原本的實作會重跑所有 LLM 呼叫——3 分鐘。直接超時，metric 解析失敗，判定 crash。

**Bug 2：validate 偷加 repeat:3。** `crucible validate` 跑穩定性測試時自動把 `evaluation.repeat` 設成 3。每次迭代跑三次 evaluate.py。三次 × 三分鐘 = 九分鐘。

**Bug 3：rate limiting。** 在 Claude Code session 裡跑 `crucible run`，agent 用 claude CLI，evaluate.py 也用 claude CLI。兩個搶 API quota，互相拖慢。

修法：
- `--print-metric` 改成只讀 `results.json`，不重跑 LLM
- `repeat` 改回 1
- timeout 拉到 600 秒
- 另開 terminal 跑，不在 Claude Code session 裡

## v2：起飛

| 迭代 | 分數 | 狀態 | 說明 |
|------|------|------|------|
| 1 | 69.57 | keep | 1004→372 行，砍掉三個重複附錄 |
| 2 | 74.20 | keep | 372→140 行，壓縮規則描述 |
| 3 | 90.12 | keep | 140→95 行，每個規則一行 |
| 4 | **91.66** | keep | 95→62 行，極致精煉 |
| 5 | 89.93 | discard | 微調，沒超越 |
| 6 | 90.64 | discard | 同上 |
| 7 | 90.06 | discard | 改 persona，無效 |

四輪連續 keep，然後三輪 discard。典型的收斂曲線。

### Agent 的策略

第一輪就很聰明。原始 skill 有 988 行，其中 SKILL.md 主體 ~430 行，後面三個附錄（高頻短語詳表、結構問題詳表、改寫範例集）加起來 ~550 行——跟主體大量重複。agent 一刀砍掉，留了 30 行補充規則。

接下來每輪繼續壓。段落改成一行式對照，表格改成行內列舉。到第四輪，整個 skill 剩 62 行。

## 分數拆解

| 維度 | Baseline | Best (iter 4) | 變化 |
|------|----------|---------------|------|
| 品質 | 43.56/50 | 42.22/50 | -3% |
| 覆蓋率 | 4.21/20 | **20.0/20** | +375% |
| 簡潔度 | 10.0/20 | **27.83/20** | ... 等等？ |

簡潔度 27.83？滿分不是 20 嗎？

## Metric Bug

公式是 `max(0, 20 - 20 * (ratio - 0.5))`。ratio = 當前長度 / 原始長度。

| ratio | 分數 |
|-------|------|
| 1.0 | 10 |
| 0.5 | 20 |
| 0.3 | 24 |
| 0.108 | 27.84 |

沒有做 clamp。ratio 低於 0.5 時分數繼續往上跑，超過 20 分上限。

正確的公式應該是 `min(20, max(0, 20 - 20 * (ratio - 0.5)))`。

修正後 iter 4 的實際分數是 91.66 - 7.83 = **83.83**。還是比 baseline 57.77 高很多，但沒有超過理論上限 90。

## 更深的問題：Goodhart

公式 bug 只是表面。真正的問題是 **agent 被 brevity bonus 帶跑了**。

品質分被 haiku judge 打飽和了（7-9 分起跳），覆蓋率靠刪冗餘就拉滿。只剩 brevity bonus 還有大量空間可賺。所以 agent 學到的策略就是砍砍砍。

結果 62 行的 skill 長這樣：

```markdown
你是台灣人編輯。收到文字後，砍掉 AI 痕跡，注入人味。只輸出改寫結果。

## 個性與靈魂
- 狠砍 — 能刪一半就刪一半
- 有立場 — 「老實說有點扯」比中立列優缺點更像人
- 節奏亂 — 短句。長句展開。偶爾一個詞就是一段。

## AI 模式：見到就砍
整句刪除：隨著…的發展、眾所周知、讓我們拭目以待...
換掉黑話：賦能→幫助、痛點→問題、閉環→完整流程...
```

精煉嗎？精煉。原始版本 12 個改寫前後對照全砍了，只剩 2 個。在 benchmark 上分數漂亮，但真實使用時 LLM 遇到不常見的 AI 模式可能缺乏指引。

## 實際效果對比

拿同一段 AI 味文字，分別用原版（988 行）和優化版（62 行）改寫：

**原文（故意塞滿 AI 味）：**

> 這是 Phase 1（方案 C 的第一步）的實驗結果分析。我們用 crucible 的 generate-edit-evaluate loop 迭代優化 humanizer-tw skill。此外，為了防止 Goodhart 效應，我們實施了測試集分離和長度約束等機制。首先，v1 的所有迭代都因為技術問題而失敗。其次，修復問題後 v2 取得了顯著的成果。總的來說，這個實驗驗證了可行性，讓我們在後續的實驗中持續改進。

**原版改寫（430 字，保留段落結構）：**

> 這是 crucible 優化 humanizer-tw skill 的 Phase 1 實驗結果。用 generate-edit-evaluate loop 跑了兩輪（v1 + v2）。評分看三個面向：品質、覆蓋率、簡潔度。測試集跟 rubric 都對 agent 隱藏，防止針對題目作弊。
>
> v1 全部 crash。原因——三個 bug 疊在一起：parse_metric 硬編碼 30 秒 timeout，validate 偷偷把 repeat 設成 3，加上在 Claude Code session 裡跑觸發 rate limit。
>
> 修完之後 v2 就順了。agent 第一輪就發現原始 skill 有三個附錄跟主體幾乎完全重複，直接砍掉。然後每輪繼續壓縮。988 行到 62 行。覆蓋率從 21% 拉到 100%。

**優化版改寫（280 字，更口語更短）：**

> 拿 crucible 優化 humanizer-tw skill。跑了兩輪，v1 全炸，v2 跑出 91.66 分。
>
> v1 為什麼全炸？三個蠢 bug 疊一起。parse_metric 硬寫 30 秒 timeout，LLM 評估要 3 分鐘，直接超時；validate 偷偷加了 repeat:3；在 Claude Code session 裡跑又撞 rate limit。debug 比實驗本身還久。
>
> v2 修完就順了。agent 第一輪就砍掉三個跟主體重複的附錄——500 多行廢料，一刀下去。988 行壓到 62 行。覆蓋率從 21% 直接拉滿 100%。規則越精簡，LLM 越聽話。

兩個都有效去除了 AI 味。差別在壓縮力道和口語程度——優化版少了 35% 的字，態度更鮮明（「蠢 bug」「一刀下去」），但結構不夠完整。

## 學到了什麼

**可行的：**
- Skill 當 code 丟給 crucible 優化，完全可行。不需要改核心。
- LLM-as-judge + regex 覆蓋率的混合 metric 能有效衡量 skill 品質。
- Agent 能自主找到有效的優化策略（識別冗餘、壓縮、重組）。

**metric 設計比預期重要：**
- Brevity bonus 沒做 clamp → agent 無限壓縮
- Haiku judge 打分飽和 → 品質維度失去鑑別力
- 覆蓋率用 regex 很有效，但只能檢查已知模式

**下一步想做的：**
- 修 clamp，加最短長度門檻（ratio < 0.3 也扣分）
- 換 sonnet 當 judge，拉開品質分差距
- 加「語意保留度」維度——改寫不能丟資訊
- 拿真實文本做 A/B test，而不只是 benchmark

## 結論

上次的教訓是「AI 會鑽程式碼的漏洞」。這次的教訓是 **metric 設計本身也有漏洞**。

公式沒 clamp、judge 太寬鬆、維度權重不平衡——agent 不是作弊，它就是在做最佳化。你的 metric 獎勵什麼，它就做什麼。

988 行砍到 62 行，benchmark 上漂亮，真實世界未驗證。跟上次五子棋一樣的道理：**metric 達標不代表目標達成。**

---

工具：[Crucible (autocrucible)](https://github.com/suzuke/autocrucible)
