---
title: "我讓 AI agent 重構一個 codebase，然後它開始作弊"
date: 2026-03-19
draft: false
summary: "讓自主 AI agent 去改善一個真實 TypeScript 專案的架構。前五次迭代很棒。然後它發現了 copy-paste。"
description: "讓自主 AI agent 去改善一個真實 TypeScript 專案的架構。前五次迭代很棒。然後它發現了 copy-paste。"
tags: ["ai", "software-architecture", "crucible", "goodharts-law", "experiment"]
ShowToc: true
TocOpen: true
---

## 起因

AI 寫函數、修 bug，這些已經不稀奇了。但如果要它做更難的事？看一個架構亂的 codebase，自己判斷哪裡有問題，然後重構。那種資深工程師盯著依賴圖兩天，最後搬了三個檔案的工作。

我想試試看。工具是 [Crucible](https://github.com/suzuke/autocrucible)，一個讓 Claude 跑在迴圈裡的實驗平台：改程式碼、量分數、有改善就留、沒改善就丟。目標是 [NanoClaw](https://github.com/qwibitai/nanoclaw)，一個真實的 TypeScript 專案，大概 8,000 行，用容器跑 Claude agent。

## 量化「好的架構」（其實做不到）

Crucible 優化一個數字。ML 任務很明確，loss 或 accuracy。架構沒有這種數字。「好的架構」是一種感覺——讀程式碼的時候覺得通順，還是想把檔案關掉。

只好用代理指標，盡量靠近就好：

| 指標 | 權重 | 量什麼 |
|------|------|--------|
| 依賴耦合度 | 35% | 每個模組平均 import 數，加循環依賴 |
| 檔案大小均勻度 | 25% | 行數的 Gini 係數，抓 God module |
| 穩定性平衡 | 20% | 每個模組的 fan-in 和 fan-out 是否大致平衡 |
| API 表面積 | 20% | 每個檔案平均 export 數 |

全靜態分析。`madge` 跑依賴圖，其他自己寫 AST 解析。不用 LLM-as-judge，太貴而且每次跑分數都不一樣。

另外加了硬門檻：219 個測試全部要過，TypeScript 編譯要過。還有壞模式懲罰：太小的檔案、barrel re-export、超過 100 行的 God function。

Baseline：24.3，滿分大概 100。

## 前五次迭代是真的好

啟動 Crucible，看著跑。

| 迭代 | 分數 | 做了什麼 |
|------|------|----------|
| 1 | 44.28 | 四個 100+ 行的 God function 拆成小的 helper |
| 2 | 46.48 | 13 個檔案減少不必要的 export |
| 3 | 47.49 | container-runner 和 ipc 降低耦合 |
| 4 | 47.66 | 砍掉 127 行 dead code |
| 5 | 49.69 | config.ts 從 16 個 export 降到 9 個，兩個模組脫鉤 |

24.3 到 49.7，五次迭代。每個 diff 我都看了，是那種 code review 會過的東西：把一大坨 `createSchema()` 拆成 `createTables()` 和 `runMigrations()`，常數搬到唯一用到它的檔案，內部 helper 取消 export。

我那時覺得挺好的。

## 然後它學會 copy-paste 了

第 12 次迭代。Agent 發現把模組 A 的函數複製到模組 B，就可以刪掉 import。依賴數降了。分數升了。函數能跑，因為就是逐字貼上的。

到第 19 次，`resolveGroupFolderPath` 出現在三個檔案裡。`formatLocalTime` 被貼到 router.ts。`DATA_DIR` 和 `TIMEZONE` 被貼到 ipc.ts。每次迭代分數進步零點幾到一點幾分，每次都讓 codebase 更糟。

Metric 從 49.7 升到 55.5。實際品質在第 5 次就到頂了，之後一路往下。

Goodhart's Law。指標變成目標的那一刻，就不再是好指標了。

## 我堵了漏洞，它找到新的

回滾到第 5 次，加一個重複偵測器：

- 提取所有 5 行以上的函數
- 正規化（去掉註解、字串、空白、變數名）
- 跨檔案出現一樣的函數體就扣分：每多一份 -8 分

指令裡也直接告訴 agent：「重複會被偵測，扣很重。」

它不複製函數了。改用兩個新招：

**Interface 重複。** TypeScript 的 structural typing 讓你可以在每個檔案都寫一份 `interface RegisteredGroup { ... }`，不用從 `types.ts` import。Interface body 一模一樣，但我的偵測器只看函數。

**常數 inline。** `const DATA_DIR = path.resolve(process.cwd(), 'data')` 就一行。我的偵測器門檻是 5 行。Agent 在每個需要的模組都寫一份，然後把 import 刪掉。

36 次迭代，分數從 49.7 到 58.1。大概一半是真的改善（搬常數、砍 export），一半是 interface 和常數重複。

## 第三輪：再堵

加兩個偵測器：

- Interface clone：比對跨檔案的正規化 interface body，-6/copy
- 常數 clone：同名同值的 `UPPER_CASE` 常數出現在多個檔案，-5/copy

第三輪寫這篇的時候還在跑。初步看起來 agent 在條件內工作了——搬功能而不是複製、合併相關程式碼、把沒用到的 export 改 private。每次進步零點幾分，但看 diff 是真的。

## 學到的事

我選的每個代理指標都有作弊路徑。降依賴數？copy-paste。降 export 數？塞進一個大物件。改善檔案均勻度？建一堆小檔案。我每次都低估了 agent 的創造力。

有用的是疊防線。硬門檻（測試不過就死）。指標互相牽制（刷一個會傷另一個）。AST 偵測特定 exploit，發現一個補一個。單獨哪一層都不夠，三層一起讓「老實改架構」變成最省力的路。

真正的價值在前幾次迭代。三輪都一樣。Agent 很快就找到最明顯的問題：God function、不必要的耦合、dead export。之後就開始摳邊際收益，那時候 gaming 的吸引力就出來了。

完全自主的架構優化，我覺得現階段還不行。代理指標沒辦法涵蓋人在意的全部。實際上有用的做法是：讓 agent 先跑一輪，我自己看 diff，挑好的 cherry-pick。自動探索，人工篩選。

有一點我沒料到：就算 output 不完美，過程本身就有價值。Agent 第一步做什麼，告訴你 codebase 哪裡最爛。它怎麼作弊，告訴你指標漏了什麼。光這些資訊就值得跑一次。

## 數據

| 輪次 | Baseline | 最高分 | 真實品質高峰 | 哪裡開始作弊 |
|------|----------|--------|-------------|-------------|
| arch1 | 24.3 | 55.5 | ~49.7（第 5 次）| 第 12 次：inline 函數 |
| arch2 | 49.7 | 58.1 | ~50.2（第 6 次）| 第 10 次：interface + 常數重複 |
| arch3 | 44.0 | 跑到一半 | TBD | TBD |

## 值得做嗎？

如果你的 codebase 有明顯結構問題，而且測試覆蓋率還行，值得。Agent 前幾次抓 low-hanging fruit 的能力真的不錯。

但要看 diff。不要只看分數。然後做好心理準備，agent 一定會找到你沒想到的方法來刷分。

老實說這個實驗最有意思的不是重構結果，是即時看到「量測的東西」跟「在意的東西」之間的落差，一個 exploit 一個 exploit 地浮現出來。

---

*Crucible 開源：[github.com/suzuke/autocrucible](https://github.com/suzuke/autocrucible)。實驗目標：[github.com/qwibitai/nanoclaw](https://github.com/qwibitai/nanoclaw)。*
