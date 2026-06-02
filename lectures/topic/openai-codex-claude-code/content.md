# OpenAI Codex 及 Claude Code
> 這份講義不是在比「誰比較強」，而是在教你怎麼把 AI coding agent 放進真實工作流，讓它能寫、能改、能驗證，也能被你接住。

[summary]
- 不是比工具名氣 | 是比誰更適合你的工作流
- Codex 強在 OpenAI 生態系與多場景 coding agent | Claude Code 強在 terminal-first 與 repo 操作
- 真正的關鍵 | 不是 prompt 華麗，而是任務切分、驗收標準與回退設計
[/summary]

## 課程目標

### 這堂課要帶走什麼
- 理解 coding agent 在開發流程中的位置
- 分辨 OpenAI Codex 與 Claude Code 的使用情境
- 建立能落地的 prompt 與驗收方式
- 讓 AI 先做一版，但最後仍由人類負責

### 學習順序
[flow]
1. 先建立共同語言
2. 再做產品定位對照
3. 接著看實際工作流
4. 最後演練 prompt 與驗收
[/flow]

## 先搞懂問題

### 從聊天到代做
傳統聊天式 AI 很擅長回答問題，但真正的開發工作需要更多事情：
- 讀懂 repo
- 釐清限制
- 動手修改
- 跑驗證
- 在失敗時回退

這就是 coding agent 的價值。它不只是回一句建議，而是能進到工作流裡，幫你完成一段可交付的工作。

### 兩句話定義
> **OpenAI Codex**
> OpenAI 的 coding agent，官方網站把它放在「build and ship faster」與各種實際工作場景裡，包含 code review、升級 API、處理多步驟任務與協作工作流。

> **Claude Code**
> Anthropic 的 agentic coding tool，官方文件直接把它描述成「活在 terminal 裡」的工具，能協助你從想法到程式碼，並且處理 debug、導航 codebase、自動化瑣事。

### 課堂判斷題
> **先不要問誰最強，要先問誰更符合你的工作方式。**

如果你的工作型態更像：
- 需要大量 review、前端改版、API 升級與多步驟協作
- 希望接上 OpenAI 生態系

那 Codex 會是很自然的選擇。

如果你的工作型態更像：
- 以終端為核心
- 常常直接在 repo 裡跑指令、debug、修 bug

那 Claude Code 會更順手。

### 你先不要問誰最強
先問三件事：
- 你的任務是不是可以拆成明確步驟？
- 你的產出是不是可以驗證？
- 你能不能接受 agent 先做一版，再由你審核？

如果答案大多是 yes，那你就已經站在正確的使用姿勢上了。

## OpenAI Codex

### 適合的場景
OpenAI 官方的 Codex use cases 透露了一個很清楚的方向：它不是只做單點補碼，而是面向真實工作流。常見場景像是：
- GitHub PR code review
- 前端或網頁功能改動
- 升級 API 整合
- 長時間、多步驟的工程任務
- 從 Slack 或訊息串接手任務

### 我會怎麼教新人用 Codex
1. 先讓它讀懂 repo
2. 先要求它列出計畫
3. 再要求它做最小修改
4. 每一步都要有驗收方法
5. 最後一定要有人類審查

### Codex 任務模板
```prompt [label="Codex 任務模板"]
請先讀懂這個 repo，再處理以下任務：

目標：
- 

限制：
- 不要破壞既有測試
- 盡量維持現有設計風格
- 若有風險，先提出來再動手

交付物：
1. 修改計畫
2. 實際變更
3. 驗證方式
4. 可能風險與回退方案
```

### 講者觀點
> **Codex 最值得教的，不是某個按鈕，而是「把任務切到可驗證」這件事。**

如果你把需求寫成「幫我改好」，agent 很容易變成黑盒。
如果你把需求寫成「先讀 repo、列計畫、只改這幾個檔案、用這些命令驗證」，它就會更像一個可靠的工程夥伴。

[tags]
- [blue] OpenAI 生態系
- [green] Review-driven
- [orange] 長任務
- [purple] 可驗證交付
[/tags]

## Claude Code

### 適合的場景
Claude Code 官方文件把它定位成終端裡的 agentic coding tool，所以它特別適合：
- 終端工作流
- 直接在 repo 裡跑指令
- debug 與修 bug
- 進行長迭代 refactor
- 自動化瑣碎但高頻的開發工作

### 我會怎麼教新人用 Claude Code
1. 在專案根目錄啟動
2. 先要求它說明自己理解了什麼
3. 要它先提案，再動手
4. 修改後立即跑驗證
5. 每次大改動都要留出 review 入口

### Claude Code 任務模板
```prompt [label="Claude Code 任務模板"]
請以終端工作流方式處理這個專案。

先做三件事：
1. 用你的話說明你理解了什麼
2. 提出修改計畫
3. 說明你會用哪些命令驗證

接著開始修改，但每完成一段都要回報：
- 改了什麼
- 為什麼這樣改
- 怎麼驗證
- 是否需要我確認下一步
```

### 講者觀點
> **Claude Code 很適合教「在終端裡合作」這件事。**

它的優勢不是讓你少看 code，而是讓你更快把 code 變成一個可執行的工作循環。

[tags]
- [blue] Terminal-first
- [green] Commandable
- [orange] Debug friendly
- [purple] Workflow-centric
[/tags]

## 怎麼選

| 情境 | 更適合 Codex | 更適合 Claude Code |
|---|---|---|
| 你要帶學員理解 OpenAI 生態系 | 是 | 否 |
| 你要示範終端裡的實作節奏 | 否 | 是 |
| 你要跑 PR review、前端改版、API 升級 | 是 | 也可 |
| 你要讓學員感受 terminal-first 協作 | 也可 | 是 |
| 你要做長任務、反覆驗證 | 是 | 是 |

### 一句話記法
- Codex 偏向「OpenAI 系統裡的 coding agent 思維」
- Claude Code 偏向「終端裡的 agentic 開發工作流」

### 這裡最容易踩雷
- 把 agent 當搜尋引擎
- 不給清楚的驗收標準
- 不要求它先讀 repo
- 讓它一次改太大
- 沒有保留回退方案

[flow]
1. 先界定任務類型和風險
2. 選 Codex 或 Claude Code
3. 要它先讀 repo，再拆解計畫
4. 讓它產出修改、測試與驗證
5. 人類做最後審查與合併
[/flow]

## 教學設計

### 20 分鐘教案
1. 3 分鐘建立 coding agent 的基本概念
2. 5 分鐘介紹 Codex 的定位與場景
3. 5 分鐘介紹 Claude Code 的定位與場景
4. 3 分鐘完成對照表與選型
5. 4 分鐘示範 prompt、修改與驗證

### 講授重點
- 工具名稱不是重點，工作流才是重點
- 好 prompt 的核心不是花俏，而是清楚
- 能交付的輸出一定包含驗證方式
- 一定要保留人類 review 與回退

### 現場示範順序
- 先示範「列計畫」
- 再示範「做最小修改」
- 再示範「跑驗證」
- 最後示範「人工 review」

### 這門課最想帶走的三件事
- AI agent 不是萬能，但很適合處理可切分的工程工作
- 好 prompt 不是漂亮，而是可驗證
- 真正的專業是把 agent 放進流程，而不是把流程交給它

## 對照表

| 面向 | OpenAI Codex | Claude Code |
|---|---|---|
| 工作入口 | 偏向 OpenAI 生態系與協作式任務 | 偏向 terminal-first 工作流 |
| 常見任務 | review、升級、前端改版、多步驟任務 | debug、repo 操作、命令式協作 |
| 操作風格 | 先理解任務，再進行有驗收的改動 | 先在終端建立上下文，再逐步修改 |
| 教學重點 | 任務切分、回饋、驗證 | 命令節奏、追蹤、修正 |
| 適合的學員 | 想把 AI 納入工程協作流程的人 | 習慣在終端完成開發的人 |

## 實作練習

- [ ] 找一個你熟悉的 repo
- [ ] 寫下 1 個適合 Codex 的任務
- [ ] 寫下 1 個適合 Claude Code 的任務
- [ ] 為每個任務寫出驗收標準
- [ ] 為每個任務寫出回退方案
- [ ] 讓 agent 先提案，再動手

### 練習提示
> **如果任務不能被驗收，就先不要丟給 agent。**
>
> 先把需求寫成可檢查的結果，再讓 agent 動手，這樣最容易成功。

### 小結口訣
- 先定義問題
- 再選工具
- 再拆任務
- 再設驗收
- 最後才是修改

## 參考資料

- [OpenAI Codex](https://developers.openai.com/codex/)
- [Codex use cases](https://developers.openai.com/codex/explore/)
- [GPT-5.3-Codex model page](https://developers.openai.com/api/docs/models/gpt-5.3-codex)
- [Claude Code overview](https://docs.anthropic.com/en/docs/claude-code/overview)
- [Claude Code getting started](https://docs.anthropic.com/en/docs/claude-code/getting-started)

> **結尾提醒**
> 最好的 AI 協作，不是把責任丟給工具，而是讓工具幫你更快到達你本來就知道該怎麼驗證的答案。
