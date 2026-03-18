---
title: "Autoresearch 工具比較：5 種自動跑實驗的方法"
date: 2026-03-18
draft: true
summary: "karpathy/autoresearch、pi-autoresearch、autoexp、Claude Autoresearch、Crucible 的實測比較。各自的強項和弱點。"
description: "karpathy/autoresearch、pi-autoresearch、autoexp、Claude Autoresearch、Crucible 的實測比較。各自的強項和弱點。"
tags: ["autoresearch", "ai", "autonomous-experiments", "comparison"]
ShowToc: true
TocOpen: true
---

Karpathy 3 月 6 號放出 autoresearch，兩週內就長出幾十個 fork 跟重新實作。大家想做的事都一樣——讓 AI agent 自己改 code、跑起來、看數字、決定留或丟、然後再來一輪。

但怎麼做的，差很多。我自己寫了其中一個（[Crucible](https://github.com/suzuke/autocrucible)），也花時間把其他幾個的 source code 翻過一遍。以下是我看到的東西。

## 參賽者

| 工具 | Stars | 類型 | Agent | 語言 |
|------|-------|------|-------|------|
| [karpathy/autoresearch](https://github.com/karpathy/autoresearch) | 40k | Python 腳本 | Claude Code CLI | Python |
| [pi-autoresearch](https://github.com/davebcn87/pi-autoresearch) | 2k | IDE extension | Pi | TypeScript |
| [autoexp](https://github.com/wizwand/autoexp) | 55 | Prompt 模板 | 任何 coding agent | 純 Markdown |
| [Claude Autoresearch](https://github.com/uditgoenka/autoresearch) | 1.2k | Claude Code skill | Claude Code | Shell + Markdown |
| [Crucible](https://github.com/suzuke/autocrucible) | — | CLI 工具 | Claude Agent SDK | Python |

## 架構差異

五個工具，三種架構。

**Agent 驅動迴圈**（autoresearch、pi-autoresearch、autoexp、Claude Autoresearch）：Agent 說了算。改檔案、跑程式、commit、revert，都是它自己決定的。工具負責提供基礎建設（dashboard、結果記錄、git 輔助），但整個迴圈跑在 agent 的 context window 裡面。

**Orchestrator 驅動迴圈**（Crucible）：一支 Python 程式掌控迴圈。Agent 每輪被叫來一次，只管生 code 修改。跑實驗、解析 metric、commit、revert 全部由 orchestrator 處理。Agent 碰不到迴圈本身。

這是最根本的差別。後面所有不同點，都從這裡長出來的。

## karpathy/autoresearch

原版。630 行 Python。目標很單一：在一張 GPU 上訓練 nanochat。

它做的事：讀 `program.md`，subprocess 叫 Claude Code CLI，跑訓練腳本，看 val_bpb，留或丟，寫進 `results.tsv`。乾淨，沒多餘的東西。

它不做的事：檔案存取控制、metric 驗證、多任務、session recovery。它是一支針對特定 workflow 的腳本，在那個 workflow 上跑得很順。

4 萬顆星是因為概念，不是因為 code。大部分 fork 就是改改硬體支援（MLX、Windows、CPU）或者把它從 nanochat 解綁。

## pi-autoresearch

Shopify CEO Tobi Lütke 拿它來優化 Liquid 模板引擎：parse+render 快了 53%，記憶體配置少了 61%，跑了大概 120 個自動實驗。靠這個紅的。

1,888 行 TypeScript，是一個 Pi IDE extension。三個工具：`init_experiment`、`run_experiment`、`log_experiment`。Agent 透過呼叫這些來推動迴圈。

好的地方：
- **Dashboard UI**，即時狀態 widget 加全螢幕 overlay。你可以盯著實驗跑。
- **Backpressure checks**，靠 `autoresearch.checks.sh`。每次 benchmark 跑完可以再跑 tests/lint/types。checks 沒過就不准 keep。擋掉「快但爛」的優化。
- **Keep/discard 靠程式碼執行。** `log_experiment` 收到 status=discard 就自動 git revert。這很可靠。
- **Session recovery。** `autoresearch.md` 的設計目標是讓一個全新 agent 讀完就能接手。
- **Ideas backlog。** `autoresearch.ideas.md` 記下有潛力但還沒試的方向。

問題：
- **綁死 Pi。** 沒有 Pi IDE 就沒戲。沒有 CLI，不支援別的 agent。
- **檔案保護只靠文字。** Skill 裡面寫「Off Limits: 不要動這些檔案。」Agent 要動？沒東西攔它。
- **沒有 metric 驗證。** Agent 報什麼數字就記什麼數字。

## autoexp

極簡路線：三個 markdown 檔，零行程式碼。`make_autoexp.md` 叫 agent 掃你的 repo、產一份客製的 `autoexp_program.md`，然後你叫 agent 照著跑就好。

好的地方：
- **零設定。** 一句 prompt 就開工。不用裝、不用寫 config、沒有依賴。
- **不挑 agent。** Claude Code、Codex，任何讀得懂 markdown 的 coding agent 都行。
- **模板品質不錯。** `program_template.md` 涵蓋了 baseline、keep/discard、failure handling、branching。

問題：
- **全靠 prompt。** 沒 timeout、沒檔案保護、沒 metric 驗證、沒 git 自動化。Agent 全靠自律。
- **結果追蹤也靠 agent 自己。** TSV log 是 agent 自己寫的。它忘了或格式不對，你就丟資料。
- **沒辦法 resume**，只能「讀 markdown 然後自己猜接到哪了。」

autoexp 回答的問題是：autoresearch 最精簡能到什麼程度？答案是一個好 prompt。夠不夠用嘛，看你多信任你的 agent。

## Claude Autoresearch (uditgoenka)

Claude Code skill，把 autoresearch 泛化到 ML 之外。有 `/autoresearch`（主迴圈）、`/autoresearch:plan`（設定精靈）、`/autoresearch:security`（安全審計）、`/autoresearch:debug`、`/autoresearch:fix`、`/autoresearch:ship`。

好的地方：
- **一開始就打算通用。** 核心原則文件明確寫了適用 code、marketing、sales、DevOps，只要有 metric 就行。
- **Guard 指令。** 跟 metric 驗證是分開的。Guard（像 `npm test`）必須永遠通過，防止你在衝 metric 的時候搞壞其他東西。Crucible 沒有這個。
- **原則文件寫得好。** 從 Karpathy 版本提煉出 7 條核心原則：「約束 = 啟動器」、「metric 必須可機械化」、「驗證要快」。
- **有限迭代。** `Iterations: N` 跑完 N 輪自動停。放著過夜跑很方便。

問題：
- **一樣是 agent 驅動的限制。** 迴圈在 context window 裡。檔案保護靠 prompt。Git 是 agent 自己管的。
- **功能太多。** 6 個子指令、STRIDE/OWASP 安全審計、ship workflow。對一個「改、跑、看、留/丟」的工具來說，表面積大了。
- **只支援 Claude Code。**

## Crucible (autocrucible)

先講清楚：這是我做的。我盡量公平地談 tradeoff。

好的地方：
- **護欄是程式碼強制的。** 檔案存取控制（editable/readonly/hidden）透過 Claude Agent SDK 的 `PreToolUse` hooks 執行。Agent 物理上讀不到 hidden 檔案。不是「請不要看 evaluate.py」，是「SDK 在 Read tool 跑之前就攔掉了」。
- **Orchestrator 控制迴圈。** Agent 不可能跳過迭代、忘記 commit、revert 錯東西。
- **Metric 驗證。** 程式碼直接拒絕 NaN、Inf 和垃圾值。
- **工具白名單。** Agent 手上只有 Read、Edit、Write、Glob、Grep。沒有 shell、沒有 subprocess、不能上網。
- **Git 管理。** 每次 run 開一條 branch，失敗的打 tag，結果自動記錄。

問題：
- **沒有 dashboard。** 就是 CLI。`crucible status` 和 `crucible history` 有，但不是即時 UI。
- **設定成本比較高。** 要寫 `config.yaml` + `program.md` + `evaluate.py`。autoexp 一句 prompt 就搞定。
- **只追蹤一個 metric。** 沒有 secondary metrics。約束得自己寫進 evaluate.py。
- **沒有 backpressure checks。** pi-autoresearch 的 `checks.sh` 是好概念，Crucible 沒做。
- **只支援 Claude Agent SDK。**
- **迭代比較慢。** Orchestrator 本身有開銷，加上每輪都要走一次完整的 SDK call。

## 真正的比較

| | autoresearch | pi-autoresearch | autoexp | Claude Autoresearch | Crucible |
|---|---|---|---|---|---|
| **迴圈控制者** | Agent | Agent | Agent | Agent | Orchestrator |
| **檔案保護** | 無 | Prompt | Prompt | Prompt | 程式碼 (SDK hooks) |
| **Metric 驗證** | 無 | 無 | 無 | 無 | 程式碼 (拒絕 NaN/Inf) |
| **Keep/discard** | 腳本 | 程式碼 | Prompt | Prompt | 程式碼 |
| **Git 管理** | 腳本 | 程式碼 (auto-revert) | Prompt | Prompt | 程式碼 (branch/tag/revert) |
| **Dashboard** | 無 | 有 (TUI) | 無 | 無 | 無 |
| **設定難度** | 改 program.md | /autoresearch X | 一句 prompt | /autoresearch:plan | 寫 config + evaluate.py |
| **Agent 平台** | Claude CLI | Pi | 任何 | Claude Code | Claude Agent SDK |
| **Backpressure checks** | 無 | 有 | 無 | 有 (guard) | 無 |
| **適用範圍** | 只有 ML | 任何 metric | ML (模板) | 任何 metric | 任何 metric |

## 你該用哪個

**第一次想玩 autoresearch？** 用 autoexp 或 Claude Autoresearch。一分鐘內就能跑起來。

**想要好看的介面、即時 dashboard？** 用 pi-autoresearch（前提是你用 Pi）。用 Claude Autoresearch（前提是你用 Claude Code）。

**要跑一整夜，結果不能出差錯？** 用 Crucible。Orchestrator 不會忘記 commit。Agent 沒辦法碰評估程式。Metric 有程式碼驗證。

**就是要跑原版 nanochat？** 直接用 karpathy/autoresearch，它就是為那個寫的。

怎麼選，看你在意什麼。在意上手速度，走 prompt 方案。在意結果可靠，走程式碼強制方案。多數人從前者開始，然後在 agent 幹了什麼怪事之後，轉向後者。

## 實測：同一個任務，四種工具

我用同一個排序優化任務測了四種工具。同樣的起始 `sort.py`（Python 內建 `arr.sort()`）、同樣的 `benchmark.py`、同樣的 metric（ops/sec，越高越好）。每個跑約 5 個迭代。

四個測試都在 Claude Code Max 底下用 Claude Opus 4.6 跑。同樣的模型、同樣的訂閱，推理能力沒有差異。唯一的變數是工具架構和 agent 能看到什麼。

Crucible 把 benchmark.py 設為 hidden，agent 看不到。其他三個都看得到。

### 結果

| 工具風格 | Baseline | 最佳 ops/sec | 改善 | 迭代 | 看得到 benchmark? |
|---------|----------|-------------|------|------|--------------------|
| autoresearch | 92.80 | 276.79 | +198% | 4 | 看得到 |
| autoexp | 97.55 | **322.94** | +231% | 6 | 看得到 |
| Claude Autoresearch | 98.55 | 300.11 | +205% | 5 | 看得到 |
| Crucible | ~92 | 292.40 | +218% | 5 | **看不到** |

### 各 agent 做了什麼

**autoresearch 風格**（4 迭代，最佳 276.79）：

| 迭代 | ops/sec | 狀態 | 說明 |
|------|---------|------|------|
| 0 | 92.80 | baseline | Python 內建 sort |
| 1 | 214.37 | keep | numpy sort |
| 2 | 161.06 | revert | numpy stable sort，反而更慢 |
| 3 | 276.79 | keep | numpy int32 |

**autoexp 風格**（6 迭代，最佳 322.94）：

| 迭代 | ops/sec | 狀態 | 說明 |
|------|---------|------|------|
| 0 | 97.55 | baseline | Python 內建 sort |
| 1 | 213.77 | keep | numpy sort |
| 2 | 244.89 | keep | numpy fromiter + quicksort |
| 3 | 167.99 | revert | numpy stable sort（radix），更慢 |
| 4 | **322.94** | keep | numpy int32 quicksort |
| 5 | 300.64 | revert | np.array int32 quicksort，稍慢的變體 |

**Claude Autoresearch 風格**（5 迭代，最佳 300.11）：

| 迭代 | ops/sec | 狀態 | 說明 |
|------|---------|------|------|
| 0 | 98.55 | baseline | Python 內建 sort |
| 1 | 222.84 | keep | numpy sort |
| 2 | 244.94 | keep | numpy fromiter + quicksort |
| 3 | **300.11** | keep | 自己寫了一個 C radix sort，編譯成 .dylib，用 ctypes 呼叫 |
| 4 | 87.32 | revert | 改用 ctypes array，反而更慢 |

**Crucible**（5 迭代，最佳 292.40）：

| 迭代 | ops/sec | 狀態 | 說明 |
|------|---------|------|------|
| 1 | 206.22 | keep | 換 numpy introsort（C/SIMD） |
| 2 | 271.72 | keep | int32 取代 int64，減半頻寬 |
| 3 | 277.01 | keep | 預分配 numpy buffer |
| 4 | 264.70 | discard | struct.unpack，更慢 |
| 5 | **292.40** | keep | array.array 零拷貝橋接 numpy |

### 迭代次數的問題

我沒預料到的事：我跟四個 agent 都說「跑 5 個迭代」。它們各自的解讀不一樣。

- **autoresearch**：跑了 4 次（把 baseline 算成第 1 次，所以「5」= baseline + 3 個實驗）
- **autoexp**：跑了 6 次（把 baseline 算成第 0 次，然後跑了 5 個實驗）
- **Claude Autoresearch**：跑了 5 次（剛好對）
- **Crucible**：跑了 5 次（`--max-iterations 5` 寫在程式碼裡，沒有模糊空間）

三個 agent，同樣的指令，三種理解。Crucible 的 orchestrator 不會有這問題，因為 Python 的 for 迴圈數到 5 就是 5。聽起來是小事，但如果你放著跑 100 個迭代過夜，「差一」就不是小事了。

### 這告訴我們什麼

autoexp 的 agent 拿到最高分（322.94）。Claude Autoresearch 的 agent 最有創意——它直接寫了一個 C library 然後編譯。Crucible 的分數不是最高。

但看看 Claude Autoresearch 的 agent 做了什麼：寫 C radix sort、編譯成 shared library、用 ctypes 載入。它能這樣做是因為它有 shell 權限，而且能讀 benchmark 來了解輸入大小和資料型態。Crucible 的 agent 沒有 shell，也看不到 benchmark。它只能從第一原理出發。

這重不重要，看你的場景。如果目標是「不擇手段把這段 code 搞快」，讓 agent 看到一切、給它 shell。如果目標是「找到能泛化的改進，而不是針對特定 benchmark 的 fitting」，就隱藏 evaluation。

四個工具的測試也證實了一件事：在乖巧的任務上，prompt 方案跟程式碼方案表現差不多。排序 benchmark 很單純，metric 很誠實，agent 沒有理由作弊。工具之間的差異只有在任務變得對抗性的時候才會浮現（見我[前一篇關於 agent 作弊的文章](/blog/zh-tw/posts/ai-cheating-experiments/)）。
