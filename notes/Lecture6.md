# Lecture 6：Agent Design Patterns & Scaffolds Agent 設計模式與鷹架

- 日期：10/12（第 4 週）
- 講者：Diyi Yang、Michael Ryan、John Yang
- 投影片：[lecture06.pdf](../slides/lecture06.pdf)（86 頁）
- 導讀：[Lecture 6 逐頁導讀](Lecture6-guide.html)
- 必讀：[ReAct](http://127.0.0.1:8329/reading/Yao2023_ReAct_ICLR2023.html)
- 搭配閱讀：Building Effective Agents（Part 1–2 的圖都出自這篇）、Effective Context Engineering（Part 4）

**學習目標**（p.6）：
1. 定義 workflow 和 agent 常見的設計模式
2. 比較各種管理 agent 上下文的模式
3. 分析一個實際上線的 agent harness（Pi）裡做了哪些設計決策

**整講的主軸**：控制流程交給程式還是交給模型？每一種模式都是在這兩端之間找平衡。開場引用 Martin Fowler（p.4）：「模式都是半成品，你永遠得自己把它完成，並調整到適合你的環境。」所以這講教的是可以改的範本，不是標準答案。

| 段落 | 頁數 | 時間 |
|---|---|---|
| 1. 回顧 workflow vs agent | p.8–12 | 5 分 |
| 2. Workflow 設計模式 | p.13–30 | 15 分 |
| 3. Agent 模式：推理與行動 | p.31–54 | 20 分 |
| 4. Agent 模式：上下文管理 | p.55–75 | 20 分 |
| 5. 案例：Pi | p.76–83 | 10 分 |

## 閱讀清單

| 類別 | 文章 | 本站譯文 | 一句話 |
|---|---|---|---|
| **必讀** | [ReAct: Synergizing Reasoning and Acting in Language Models](http://127.0.0.1:8329/reading/Yao2023_ReAct_ICLR2023.html) | 全譯對照 | 讓 LLM 交錯產生「思考」與「動作」：推理引導行動，行動帶回證據。在 HotpotQA、FEVER 上減少幻覺與錯誤傳播；在 ALFWorld、WebShop 上只用 1–2 個示例，就分別比模仿學習／強化學習高 34%、10%。 |

> 必讀一定要讀；推薦、補充可以依時間和興趣挑。清單依本站 index.html 的閱讀材料整理。

---

## 1. 回顧：Workflow 與 Agent（p.8–12）

差別只有一個：**控制流程在程式手上，還是在模型手上**。

p.9 是 Lecture 5 那張光譜圖的簡化版：

| | 模型 Model | Workflow／複合 AI 系統 | Agent |
|---|---|---|---|
| 角色 | 大系統裡的一小塊 | 邏輯和編排寫在程式裡 | 控制流程交給 AI 模型 |
| 例子 | GPT-6 Sol、Claude Fable 5.1 | email 分類器、新聞摘要器 | 客服 agent、coding agent |

p.10：workflow 的控制流程在程式邏輯裡，所以**比較好預測**；agent 由 LLM 決定，所以**比較能適應、比較彈性**。

p.12 對照表：

| | Workflow | Agent |
|---|---|---|
| 控制流程 | 程式 | LLM |
| 如何應變 | 事先寫死的分支、路由和迴圈 | 依觀察結果決定下一步 |
| 適合 | 結構清楚、會重複的任務 | 需要探索和彈性判斷的任務 |
| 例子 | 分類客服單 → 撈出相關規定 → 草擬自動回覆 | 用工具調查並解決客服單上的問題 |

真實系統常常同時用到 workflow 和 agent。後面的 STORM、ReWOO、Pi 都是例子。

---

## 2. Workflow 設計模式（p.13–30）

### 2.1 五種基本模式（p.14–20）⭐

出自 Anthropic〈Building Effective Agents〉。

| 模式 | 做法 | 什麼時候用 |
|---|---|---|
| 提示鏈 Prompt Chaining | 把問題拆成幾個小步驟，前一步輸出接到下一步 | 任務複雜、拆開會比較準 |
| 路由 Routing | 先分類，再交給合適的 LLM 處理 | 輸入有幾種明顯不同的類型 |
| 平行化 Parallelization | 把工作分給多條平行的呼叫 | 子任務彼此獨立，或想多跑幾次取共識 |
| 協調者–工作者 Orchestrator-Workers | 一個 LLM 找出子任務，再派給工作者完成 | 事先不知道要拆成哪些、幾個子任務 |
| 評估者–優化者 Evaluator-Optimizer | 產生輸出 → 評語 → 修改，反覆進行 | 有清楚的評分標準，而且修改確實有幫助 |

**提示鏈**（p.16）：用延遲換準確度，多了幾次呼叫，但每次只做一件簡單的事。原文例子是「先寫行銷文案，再翻譯」。

**路由**（p.17）：分完類後交給不同模型或不同 prompt。也能省錢省時間：簡單的情況交給較弱、較便宜的模型。

**平行化**（p.18）有兩種：
- **分段 Sectioning**：把獨立的工作拆開同時做，例如一邊判斷查詢是否安全、一邊處理這個查詢。
- **投票 Voting**：一種推論時擴展（inference-time scaling），把同一個輸入跑好幾次，再由彙整者（aggregator）做最後結論。

好處是讓每次呼叫只專注一個面向。

**協調者–工作者**（p.19）：和平行化很像，差別在**子任務是執行時才動態產生的**。平行化的分法是寫死在程式裡的，協調者–工作者的分法是由 LLM 決定的。

**評估者–優化者**（p.20）：一個 LLM 產生，另一個 LLM 評估並給回饋，被退回就帶著回饋重改。要成立需要：
1. 回饋要清楚、說得出來
2. 有 LLM 能給出這種回饋
3. （附帶）產生者真的能根據回饋改進

> 這幾個條件在第 3 節會再出現：Reflexion 和 CRITIC 有效，是因為回饋來自外部環境或工具；純粹讓模型自己評自己，第 2 個條件常常不成立（p.41）。

### 2.2 模式的組合：STORM（p.21–25）⭐

真實系統都是這些基本模式的組合。

**STORM 是什麼**：Stanford OVAL 做的系統，輸入一個主題（例如「2024 花蓮地震」），自動寫出一篇有目錄、有引用的**維基百科式長文**。

流程（p.23）：
1. **調查**：找主題的相關文章
2. **找觀點**：從相關文章歸納出幾種不同的視角
3. **讀與問**：扮成「維基作者」的 LLM 帶著某個視角，向「專家」LLM 發問
4. **專家回答**：把問題拆成查詢 → 搜尋並篩選 → 綜合出答案；可信來源加進參考文獻
5. **直接生成**：只靠模型知識先寫出草稿大綱
6. **精修**：用所有對話紀錄把草稿大綱修成最終大綱

p.24 把流程對到五種模式：

| STORM 的部分 | 對應模式 |
|---|---|
| 調查 → 大綱 → 精修，一步接一步 | 提示鏈 |
| 多個視角各自進行對話 | 平行化 |
| 依視角分派對話 | 協調者–工作者（投影片註明：其實接近平行化） |
| 作者與專家一問一答的迴圈 | 大致是評估者–優化者 |

結果（p.25）：STORM 的 ROUGE-1 45.82、實體召回 14.10，都高於最強基準 oRAG（44.26、12.57）；**拿掉大綱階段後 ROUGE-1 掉到 26.77**，可見提示鏈裡「先寫大綱」這一步很關鍵。

### 2.3 Archon：自動搜尋模式組合（p.26–29）

STORM 的組合是人設計的；**Archon 讓程式自動搜尋最好的組合**。

- 輸入：目標基準、推論呼叫預算、可用的 LLM、可用的推論時技巧。
- 用**貝氏優化**反覆提出架構、跑基準評分，最後輸出最好的架構。
- 可用的「積木」（p.27）：

| 積木 | 作用 |
|---|---|
| 產生器 Generator | 依指令產生候選回答 |
| 融合器 Fuser | 把多個候選合併成一個 |
| 評論者 Critic | 列出每個候選的優缺點 |
| 排序器 Ranker | 回傳前 K 名候選 |
| 驗證器 Verifier | 只留下推理經過驗證的候選（推理任務） |
| 單元測試產生器／評估器 | 生成單元測試，再用它替候選打分（推理任務） |

搜出來的架構例子（p.28）：4 個不同模型的產生器各取 5 個樣本 → 評論者 → 排序器 → 4 個融合器 → 驗證器 → 融合器 → 輸出。

結果（p.29）：token 預算從 10K 加到 100K，平均分數一路升到約 76%，超過 AFlow、ADAS 等自動設計方法。

> 重點不是 Archon 本身，而是它把「推論時技巧」當成可以排列組合的模組，這是 workflow 模式的延伸。概念上和 Lecture 5 的 DSPy 優化器類似：DSPy 自動調 prompt，Archon 自動調架構。

### 2.4 小結（p.30）
- Workflow 比 agent 更容易重現和除錯。
- 實務上，真正的 workflow 通常是這五種元件組成的更大系統。

---

## 3. Agent 設計模式：推理與行動（p.31–54）

Agent 迴圈（p.32）：人類 → LLM 呼叫 → 行動 → 環境 → 回饋 → LLM 呼叫 … → 停止。除了工具本身，還要設計 **LLM 怎麼推理、怎麼和環境互動**。這一段有四個關鍵想法：

| 模式 | 關鍵想法 | 今天的狀況（p.54） |
|---|---|---|
| ReAct | 先想再做 | 至今仍是標準做法 |
| Reflexion | 從失敗中反思 | 已經內建在推理模型的推理裡 |
| CodeAct | 用程式碼當行動 | 程式碼是很好的行動形式 |
| ReWOO／LLMCompiler | 先規劃再執行 | 今天主要靠平行工具呼叫實現；規劃仍然重要 |

有時仍會用 workflow 來編排「子 agent」（p.54）。

### 3.1 ReAct：先推理再行動（p.33–36）⭐

> 這一節依 ReAct 論文整理並查證；3.1.3 工程實作是延伸，論文沒有寫。記憶怎麼接進來，見 [Lecture 7 筆記](Lecture7.md)。

#### 3.1.1 投影片怎麼講（p.34–36）

p.34 用一題 HotpotQA 比較四種做法：「除了 Apple Remote，還有什麼裝置能控制 Apple Remote 原本設計要搭配的程式？」

| 做法 | 過程 | 結果 |
|---|---|---|
| 標準 | 直接回答「iPod」 | ✗ |
| 只推理 CoT | 憑記憶推理，以為是 Apple TV | ✗ 幻覺 |
| 只行動 Act | 會搜尋，但沒有推理串起來，最後答「yes」 | ✗ |
| **ReAct** | 查 Apple Remote → 知道它控制 Front Row → 查 Front Row (software) → 答「keyboard function keys」 | ✓ |

p.36（PaLM-540B）：

| 方法 | HotpotQA EM | FEVER 準確率 |
|---|---|---|
| Act | 25.7 | 58.9 |
| CoT | 29.4 | 56.3 |
| ReAct | 27.4 | 60.9 |
| ReAct → CoT-SC | **35.1** | 62.0 |
| CoT-SC → ReAct | 34.2 | **64.6** |

ReAct 比 Act 好，說明**行動前先推理有幫助**；兩者搭配最好。

#### 3.1.2 論文原本長什麼樣

**原始形式**：模型照 few-shot 示例，一行一行寫出軌跡。論文用 PaLM-540B，只靠 prompt，沒有微調。HotpotQA 的動作只有三種：

| 動作 | 作用 |
|---|---|
| `search[entity]` | 回傳該維基頁面的前 5 句；頁面不存在就建議 5 個相似的條目 |
| `lookup[string]` | 回傳頁面中下一個含該字串的句子（模擬 Ctrl+F） |
| `finish[answer]` | 結束，給出答案 |

```
Question: 哪個樂團的成立時間比較早，A 還是 B？
Thought 1: 我要分別查兩個樂團的成立年份。
Action 1:  search[A]
Observation 1: A 是……成立於 1985 年……
Thought 2: A 是 1985 年。接著查 B。
Action 2:  search[B]
Observation 2: B 是……成立於 1991 年……
Thought 3: 1985 早於 1991，所以是 A。
Action 3:  finish[A]
```

（上面是示意格式，不是論文原例；論文原例就是上面 Figure 1 的 Apple Remote 問題，Lecture 5 p.49 也有引用。）

**讀論文時要抓的五個重點**

1. **Thought 不影響環境**：Thought 是放在「語言空間」裡的動作，不會產生 Observation，只會更新上下文，幫下一步做決定（論文 §2）。
2. **兩種任務的節奏不同**：知識型任務（HotpotQA、FEVER）每一步都是 Thought → Action → Observation；決策型任務（ALFWorld、WebShop）動作很多，Thought 只**零星**出現在關鍵位置，由模型自己決定何時思考。
3. **ReAct 不是在所有任務上都贏**：HotpotQA 上 ReAct 和 CoT 互有勝負，FEVER 上 ReAct 較好。最佳做法是**兩者結合**：ReAct 在限定步數內（HotpotQA 7 步、FEVER 5 步）沒答出來，就退回 CoT-SC；CoT-SC 多數決不夠一致，就改用 ReAct。
4. **失敗模式**（HotpotQA 錯誤分析）：
   - CoT 失敗中有 **56%** 是幻覺；ReAct 因為查得到外部資料，較少幻覺。
   - ReAct 特有的錯誤是**重複產生前一步的 Thought 和 Action，跳不出迴圈**，這就是工程上一定要設最大步數的原因。
   - 搜尋沒找到有用資訊佔錯誤案例的 **23%**，一旦搜錯，模型很難救回來。
5. **小模型靠微調**：PaLM-8B/62B 只用 prompt 時，ReAct 是四種方法中最差的；但用 3,000 條正確軌跡微調後反而最好（p.35）。今天在後訓練階段微調模型扮演 agent 已經是常態。

#### 3.1.3 工程實作：Action 變成 tool calling

現在的模型 API 通常把論文裡的自由格式 Action（`search[...]`）做成**結構化的 tool calling**，由 API 保證格式，後端不用自己解析文字。

**① 定義工具**：名稱、用途、參數都要寫清楚。

```json
{
  "name": "get_article",
  "description": "根據文章 ID 取得完整文章內容",
  "parameters": {
    "type": "object",
    "properties": { "article_id": { "type": "integer" } },
    "required": ["article_id"]
  }
}
```

LLM **不會**真的執行 Python 函式，它只會產生這樣的呼叫：

```json
{ "tool": "get_article", "arguments": { "article_id": 123 } }
```

真正執行的是後端。

**② 後端迴圈**：

```python
TOOLS = {
    "search_articles": search_articles,
    "get_article": get_article,
    "summarize_article": summarize_article,
}

def react_loop(messages, max_steps=8):
    for _ in range(max_steps):                      # 對應論文的步數上限，避免重複迴圈
        response = llm.chat(messages=messages, tools=TOOL_DEFINITIONS)
        if not response.tool_calls:                 # 沒有呼叫工具 = 最終答案（相當於 finish[...]）
            return response.content
        messages.append(response.message)           # 保留模型這一步（含 tool call）
        for call in response.tool_calls:
            messages.append({
                "role": "tool",
                "tool_call_id": call.id,            # 對應是哪一次呼叫
                "content": execute_tool(call.name, call.arguments),   # Observation
            })
    return "超過最大步數，停止"                        # 不能讓迴圈無聲結束

def execute_tool(name, arguments):
    """不在白名單就拒絕；出錯不丟例外，把錯誤當 Observation 回給模型。"""
    if name not in TOOLS:
        return f"錯誤：沒有 {name} 這個工具"
    try:
        return serialize(TOOLS[name](**arguments))
    except Exception as e:
        return f"錯誤：{e}"
```

> 修正：原筆記寫成 `result = tools**arguments`，markdown 把方括號吃掉了，正確是 `TOOLS[tool_name](**arguments)`。另外 tool 訊息要帶 `tool_call_id`，迴圈跑滿步數時也要有回傳值。

**③ 跑起來的順序**

| 誰 | ReAct 角色 | 內容 |
|---|---|---|
| 使用者 | Question | 請找出資料庫內關於 Vite 8 的文章並摘要 |
| LLM | Thought + Action | 需要先搜尋 → `search_articles("Vite 8")` |
| 後端 | Observation | 執行 PostgreSQL 查詢 → 文章 ID 123、456 |
| LLM | Thought + Action | 讀最相關的那篇 → `get_article(article_id=123)` |
| 後端 | Observation | 回傳全文 |
| LLM | Thought + finish | 資訊已足夠 → 產生摘要 |

> **和論文的差別**：原生 tool calling 常常只回傳工具呼叫，**沒有寫出 Thought**。想保留 ReAct「先推理再行動」的好處（可解釋、好除錯），可以在 system prompt 要求模型呼叫工具前先用一兩句話說明理由，或使用模型的 reasoning 輸出。

### 3.2 Reflexion：把反思變成記憶（p.37–40）⭐

**關鍵想法**：自我反思有用，而且要把反思**存起來給下一次用**。

```
Actor 執行一次 → Evaluator 評分 → Self-reflection 寫反思 → 存進長期記憶 → Actor 再試
```

- 三個 LM：行動者 Actor、評估者 Evaluator、自我反思者 Self-reflection。
- **短期記憶**＝這一輪的行動軌跡；**長期記憶**＝累積下來的反思文字。
- 重複「產生軌跡 → 評估 → 寫反思 → 加進記憶」，直到評估通過或到達最大次數。
- **核心**：把評估和反思轉成行動者的「記憶」，也就是上下文。**模型權重不變，改的是下一次看到的 prompt**。

p.39 三個例子：

| 任務 | 回饋從哪來 | 反思內容 |
|---|---|---|
| 決策（ALFWorld） | 環境：沒找到鍋子 | 「以為鍋子在爐子 1，其實不在，下次去爐子 2」 |
| 寫程式 | 自己生成的單元測試沒過 | 「只檢查了括號數量、沒檢查順序」 |
| 推理（HotpotQA） | 環境回饋 0 分 | 「誤以為兩人都有多重職業」 |

結果（p.40）：
- HotpotQA 上 ReAct+Reflexion 隨嘗試次數從約 0.32 升到約 0.54；沒有反思的基準幾乎不動。
- 消融實驗：只給上一輪軌跡（EPM）0.61 → 0.66，**再加反思到 0.74**。有幫助的是 LLM 寫出的反思，不只是「再看一次過去的軌跡」。

### 3.3 自我反思的限制與 CRITIC（p.41–42）

**p.41**〈Large Language Models Cannot Self-Correct Reasoning Yet〉：**叫 LLM 評自己的回答，反而會讓表現變差**。GPT-4-Turbo 在 CommonSenseQA 從 84.0 掉到 74.5–83.5。原因是沒有新資訊進來，模型只能拿產生錯誤的同一套知識去檢查，常常把原本對的答案改錯。

**Reflexion 為什麼不一樣**：它的評估者看的是**外部環境的回饋**（測試結果、環境獎勵），不是模型自己的判斷。

**p.42 CRITIC**：讓驗證者也能用外部工具（搜尋引擎、程式直譯器、計算機、維基百科），變成「agent 式驗證者」。AmbigNQ 上 F1 隨修正輪數從約 64 升到約 75；拿掉工具只到約 67。

**今天的狀況**：這個發現早於推理模型。推理模型在推理過程中就會自我修正，所以自我反思通常內建在模型裡，不必再由外部強制。

> 💭 **我的問題：那到底該不該讓模型反思「某一步是不是做錯了」？**
>
> 看**有沒有新的外部訊號**：
> - **沒有新訊號就別叫它反思**。像「請再檢查一次你的答案」這種 prompt，p.41 證明常常有害。推理模型本來就會在推理時自己檢查，額外再要求一次多半只是浪費 token。
> - **有具體的失敗訊號才反思**：工具回傳錯誤、測試失敗、編譯錯誤、步數用完還沒完成。這時要模型寫一段「哪裡錯了、下次改怎麼做」，比直接重試好（p.40 的 0.66 → 0.74 就是這個差距）。
> - 實作上：不必每一步都反思，而是**把反思綁在失敗事件上**觸發。

> 💭 **我的問題：外部環境的回饋要怎麼給，才能讓 LLM 反思？**
>
> 外部回饋就是「LLM 以外的東西」產生的、可以客觀判斷對錯的訊號。常見來源：
>
> | 來源 | 例子 | 適合 |
> |---|---|---|
> | 執行結果 | 單元測試失敗訊息、stack trace、exit code | coding agent |
> | 靜態檢查 | 編譯器、型別檢查（tsc）、linter | coding agent |
> | 工具錯誤 | API 回 404、查詢結果為空 | 任何用工具的 agent |
> | 外部查證 | 搜尋引擎、維基百科、計算機（CRITIC 的做法） | 事實型問答 |
> | 任務評分 | 環境獎勵、標準答案比對、有評分量表的 LLM 評審 | 有明確評分標準的任務 |
>
> 放進迴圈的方式有兩層：
> 1. **最基本**：把回饋當 Observation 放回上下文。3.1.3 的 `execute_tool` 把錯誤訊息回傳給模型，就已經在做這件事；coding agent 跑 `npm test` 看到失敗，也是一樣。
> 2. **Reflexion 式**：整輪失敗後，多一個步驟要模型把失敗**整理成教訓**存起來，下一輪開頭帶著它重來：
>
> ```python
> reflections = []
> for trial in range(max_trials):
>     result = run_agent(task, extra_context=reflections)   # Actor
>     feedback = run_tests(result)                           # Evaluator：外部訊號，不是模型自評
>     if feedback.passed:
>         return result
>     reflections.append(llm(f"""任務：{task}
> 你的做法：{result.trajectory}
> 測試結果：{feedback.output}
> 用兩三句話說明哪裡錯了、下次要改什麼。"""))             # Self-reflection → 長期記憶
> ```
>
> 關鍵是 `run_tests` 那一行：**評估必須來自外部**，反思才有根據。

### 3.4 CodeAct：用程式碼當行動（p.43–46）⭐

**關鍵想法**：程式碼是行動最合適的形式。

p.44–45 用同一題比較：「在美國、日本、德國、印度中，找出買手機 CodeAct 1 最划算的國家」，有 5 個 API 可用（查匯率與稅率、查價格、換算含稅價、估運費、算最終價格）。

**JSON 行動**：每次只能呼叫一個工具，對每個國家重複 5 次，總共約 20 回合。

```json
{"tool": "lookup_rates", "country": "Germany"}
```

**程式碼行動**：一次寫完（示意，函式名稱依投影片的 5 個 API 改寫）。

```python
countries = ["USA", "Japan", "Germany", "India"]
final_prices = {}
for country in countries:
    exchange_rate, tax_rate = lookup_rates(country)
    local_price = lookup_phone_price("CodeAct 1", country)
    converted = convert_and_tax(local_price, exchange_rate, tax_rate)
    shipping = estimate_shipping_cost(country)
    final_prices[country] = estimate_final_price(converted, shipping)
most_cost_effective = min(final_prices, key=final_prices.get)
```

兩者都答日本、904.00 美元，但程式碼版行動次數少很多。優勢有兩個：
1. **控制流程和資料流**：迴圈、變數讓複雜操作變簡單，中間結果不必經過 LLM 再傳一次。
2. **直接重用現有軟體**：例如 Python 內建的 `min`、整個套件生態。

p.46：在 GPT-4、Claude 2、Gemini Pro、Llama-2 等 8 個模型上，CodeAct 的成功率較高，所需互動回合也較少。同團隊的 OpenHands 預設 agent 就是 CodeAct。

> 補充：代價是要有**沙盒**跑模型寫的程式（OpenHands 用 Docker），安全問題比 JSON 工具呼叫大。

### 3.5 先規劃再執行：ReWOO（p.47–49）⭐

**關鍵想法**：先做規劃，就能把 agent 的行動執行**變回 workflow**。

```
Planner 一次寫完計畫 → Worker 依序呼叫工具 → Solver 讀計畫＋證據作答
```

p.48 例子：「做 Hennchata 這款調酒的主要原料，是哪家干邑酒廠生產的？」Planner 一次寫出 4 步，用變數代表還沒拿到的結果：

```
#E1 = Wikipedia[The Hennchata]
#E2 = LLM[主要原料是什麼？context: #E1]       → Hennessy cognac
#E3 = Wikipedia[#E2]
#E4 = LLM[哪家酒廠生產？context: #E3]
```

Worker 依序填入證據，Solver 回答「Jas Hennessy & Co.」。

- 名字的意思是 **Reasoning WithOut Observation**：規劃時不等工具回傳，所以 LLM 不用每一步都重讀整段歷史。
- p.49：**準確度更高、token 更少**。公開基準約 0.58 → 0.61、token 約 3,600 → 1,300。
- 代價：計畫一旦寫錯，執行途中不會修正（ReAct 會）。

> 💭 **我的問題：我做重構或大範圍修改時，會先給 agent 需求背景、讓它理解專案、用 plan mode 規劃，我檢視過 plan 才開始實作。這跟 ReWOO 一樣嗎？**
>
> 精神一樣，都是「先規劃再執行」，但有三個差別：
>
> | | ReWOO | 我的 plan mode 流程 |
> |---|---|---|
> | 計畫格式 | 精確的工具呼叫序列，用 `#E1` 串起來 | 自然語言的步驟清單 |
> | 執行階段 | 照計畫機械式執行，不看觀察結果 | 每一步仍是 ReAct 迴圈，會看測試結果、讀檔再調整 |
> | 誰檢查計畫 | 沒人 | **我**，在動手之前 |
>
> 所以實際上是**混合式**：用規劃換到全局觀（重構最需要知道哪些檔案會連動），執行時保留 ReAct 的彈性。加入人工檢查是 ReWOO 沒有的：在計畫階段修正誤解最便宜，等到改了 30 個檔案才發現方向錯就很貴。
>
> 從這講可以借來改善流程的幾點：
> - **計畫裡寫驗證方式**：每一步做完要跑哪個測試或指令確認。這就是 3.3 說的外部回饋。
> - **標出彼此獨立的步驟**：可以平行做，或分給子 agent（3.6 LLMCompiler、4.6 上下文隔離）。
> - **執行時發現計畫不對，就停下來重新規劃**，不要硬做完。這是 LLMCompiler 的「動態重新規劃」。
> - 計畫存成檔案（例如 `plan.md`）也是一種**記憶卸載**（4.3）：上下文被壓縮後，agent 還能讀回計畫和進度。

### 3.6 LLMCompiler：平行執行與動態重新規劃（p.50–52）

和 ReWOO 的差別（p.50）：**平行執行**與**動態重新規劃**。

- **函式呼叫規劃器**把任務寫成一張 DAG（有向無環圖），例如：
  ```
  $1 = search(Microsoft 市值)
  $2 = search(Apple 市值)
  $3 = math($1 / $2)
  ```
- **任務擷取單元**處理依賴關係：`$1`、`$2` 沒有依賴，同時跑；`$3` 等兩者都完成。
- 結果不夠時可以**重新規劃**。

p.51 例子：「Scott Derrickson 和 Ed Wood 國籍相同嗎？」ReAct 先搜一人、再搜另一人；LLMCompiler 一開始就規劃兩個搜尋平行執行。

p.52 結果：HotpotQA 準確度 62.00%，延遲 3.95 秒，比 ReAct 的 7.12 秒**快 1.80 倍**；Movie Rec. 快 3.74 倍；ParallelQA 快 2.15 倍。準確度和 ReAct 相當或略高。

> 今天的 API 支援**平行工具呼叫**（一個回合發出多個 tool call），3.1.3 迴圈裡的 `for call in response.tool_calls` 就是在處理這種情況。LLMCompiler 的想法大部分已經內建到模型裡了。

---

## 4. Agent 設計模式：上下文管理（p.55–75）

### 4.1 上下文工程（p.56–57）⭐

p.56：agent 迴圈多了一個**上下文管理器**，每次呼叫 LLM 前，由它決定這次要放進哪些內容。

p.57 對比：
- **提示工程**：單輪問答，上下文只有系統提示和使用者訊息。
- **上下文工程**：agent 會有很多可能的內容（文件、工具、記憶檔、詳細指示、領域知識、訊息歷史），每一輪都要篩選放哪些進上下文視窗；工具結果回來後再重新篩選。

### 4.2 為什麼要管：大海撈針與兩個原則（p.58–59）⭐

p.58 大海撈針測試：在文件裡藏一個事實，要 Claude 2.1（200K）找出來。**上下文越長，找回的準確度越差**。

p.59 兩個原則：

| 聚焦上下文 Focus the Context | 保持連續 Preserve Continuity |
|---|---|
| 上下文視窗裡只留當下任務需要的東西。資訊太多就是上下文膨脹（context bloat）。 | 操作上下文時要保住需求、目標、限制、進度。實務上也代表要保住 KV Cache。 |

> 補充：KV Cache 是模型對已處理過的 token 存下的中間結果。只要上下文開頭不變，下次呼叫就能重用，省時間也省錢（API 的 prompt caching）。一旦改動前面的內容，後面的快取全部失效，所以上下文管理不能隨意重排舊內容。

### 4.3 五種上下文管理模式（p.60–61）⭐

| 模式 | 做法 | 代表例子 |
|---|---|---|
| 持久事件日誌 Persistent Event Log | 從系統事件重建上下文 | OpenHands |
| 記憶卸載 Memory Offloading | 把知識存到記憶裡，才能刪掉上下文 | MemGPT |
| 壓縮 Compaction | 把之前的上下文摘要成精簡描述 | — |
| 觀察遮蔽 Observation Masking | 隱藏或截短舊的工具輸出 | coding agent |
| 上下文隔離 Context Isolation | 交給子 agent 做特定任務，只放當下需要的 | 子 agent、skill 檔 |

### 4.4 持久事件日誌：OpenHands（p.62–63）

OpenHands 的架構（p.62）分三塊：
- **使用者介面**：命令列、Web UI、IDE 外掛
- **事件流**：記錄所有行動與觀察
- **agent 執行環境**：Docker 沙盒，含 IPython、Bash、瀏覽器

Agent 讀事件歷史 → 產生新行動 → 執行環境執行 → 產生觀察 → 寫回事件流。

p.63 重點：
- **所有事情都照實保存**，即使送給 LLM 的輸入需要改。事件日誌可以是不可變、只能附加（append-only）的。
- 每次呼叫 LLM，輸入都從事件日誌**重建**：事件清單 → 篩出可轉給 LLM 的事件 → 依 `llm_response_id` 分組 → 轉成訊息 → LLM 輸入。
- 外部觀察者可以掛 hook 監控系統狀態。

> 這個模式把「發生過什麼」（日誌）和「這次給模型看什麼」（上下文）**分開**。後面的壓縮、遮蔽都只改後者，原始紀錄不會丟。對照 3.1.3 的程式：那裡的 `messages` 同時是日誌也是上下文，一壓縮就回不去了。

### 4.5 記憶卸載：MemGPT（p.64）

把東西存到記憶或檔案，就能空出上下文、刪掉舊內容。

p.64 例子：
1. 2/7 使用者說：「今天很開心，男友 James 幫我烤了生日蛋糕。」
2. 系統警告上下文快滿（記憶壓力）→ agent **自己**呼叫 `working_context.append("Birthday is February 7")`、`append("Boyfriend named James")`
3. 之後使用者提到「我們去了六旗樂園」→ agent 呼叫 `recall_storage.search("six flags")`，找到過去「我和 James 第一次是在六旗認識的」
4. 回覆：「你是和 James 一起去的嗎？你們在那裡認識真浪漫！」

> 模型像作業系統一樣，自己決定什麼留在「記憶體」（上下文）、什麼放到「硬碟」（外部儲存）。詳見 [Lecture 7 筆記](Lecture7.md)。

### 4.6 壓縮與觀察遮蔽（p.65–68）

**壓縮 Compaction**（p.65）：讓 LLM 把之前的「推理、行動、觀察」摘要起來，用更少的 token 表示歷史。讓壓縮本身也是 agent 式的（模型自己決定保留什麼、何時壓縮）會有幫助。

**觀察遮蔽 Observation Masking**（p.66–67）：不用 LLM 摘要，**直接把舊的觀察換成佔位符或截短**，推理和行動保留原樣。

p.65 的圖比較三種上下文：

| | T₁ | T₂ | T₃（最新） |
|---|---|---|---|
| (a) 原始 | 推理＋行動＋觀察 | 推理＋行動＋觀察 | 推理＋行動＋觀察 |
| (b) 觀察遮蔽 | 推理＋行動＋~~觀察~~ Mask | 推理＋行動＋~~觀察~~ Mask | 推理＋行動＋觀察 |
| (c) LLM 摘要 | Summary 1 | ← 併入 Summary 1 | 推理＋行動＋觀察 |

**為什麼觀察遮蔽特別適合 coding agent**（p.66）：軟體工程 agent 的上下文裡，**環境觀察佔 83.9% 的 token**，推理 9.6%、行動 6.5%。觀察就是檔案內容、測試輸出、搜尋結果，又長又容易過時（檔案之後還會改），而且需要時可以再讀一次。把它遮掉省最多，又不會丟失「做過什麼、為什麼做」。

> 補充（論文結論，非投影片原文）：論文〈The Complexity Trap〉發現簡單的觀察遮蔽在成本大約減半的同時，解題率能和 LLM 摘要打平甚至略好。摘要要多花一次 LLM 呼叫，還可能漏掉細節。

### 4.7 上下文隔離與漸進揭露（p.69–70）

**上下文隔離**（p.69）：先規劃，再把工作分給 Agent 1、Agent 2，每個 agent 有各自的上下文視窗，可以平行計算。子 agent 只回傳結果，過程中讀的大量資料不會塞進主 agent 的上下文。

**漸進揭露**（p.70）：一開始不必把所有工具和上下文都放進 prompt，需要時再揭露。例子是 Anthropic 的 skill 檔（`SKILL.md`）：

```markdown
---
name: pdf
description: 處理 PDF：擷取文字、合併、填表單……
---
（平常只有上面幾行在上下文裡）

# 正文：要用時才讀
填表單的細節見 [forms.md](forms.md)   ← 正文還能再指向其他檔案
```

### 4.8 模式可以組合（p.71–75）

p.71：這些策略彼此相容。可以先把知識卸載到記憶再壓縮；也可以一邊用事件日誌建上下文、一邊截短觀察。真實系統會組合其中很多甚至全部。

p.73 LangChain 的四類分法，可以和五種模式對照：

| LangChain 分類 | 做法 | 對應本講的模式 |
|---|---|---|
| 寫出 Write | 存到長期記憶、草稿區或狀態 | 記憶卸載 |
| 選取 Select | 撈出相關工具、草稿、記憶或知識 | 漸進揭露、記憶召回 |
| 壓縮 Compress | 摘要保留相關 token；修剪移除無關 token | 壓縮、觀察遮蔽 |
| 隔離 Isolate | 在狀態中分區、放到沙盒、分給多個 agent | 上下文隔離 |

p.74 預告：記憶在 Lecture 7 細講，多 agent 在 Lecture 8 細講。

---

## 5. 案例：Pi 極簡 agent harness（p.76–83）

用前面學的模式拆解一個真實的 harness：它選了哪些、刻意不做哪些。

### 設計哲學（p.77–78）
- 口號：「Pi 是極簡的 agent harness。讓 Pi 配合你的工作流程，而不是反過來。」
- 本身非常陽春，社群在上面做了大量擴充。
- **刻意不做**：子 agent、權限檢查、規劃模式、agent 待辦清單、背景執行 bash。

### 工具與迴圈（p.79–80）
- 內建工具：read、bash、powershell、edit、write、grep、find、ls。
- **預設只開 read、bash、edit、write**。工具越少，工具說明佔的上下文越少；其他工具其實都是 bash 能做的事的子集。
- **ReAct 式迴圈**：先推理，再發出 JSON 工具呼叫。
- **上下文是一棵樹**：可以回到之前的分支，從那裡接續另一條對話。

### 上下文管理（p.81）⭐
用了第 4 節的三個模式：
1. 事件日誌轉成 LLM 輸入（持久事件日誌）
2. 透過 skill 檔做漸進揭露（上下文隔離）
3. 壓縮

LLM 實際看到的內容：

```
[system prompt] [summary（來自壓縮）] [usr][ass][tool][tool][ass][tool]  ← 從 firstKeptEntryId 開始原樣保留
```

系統提示固定在最前面（保住 KV Cache），舊對話壓成一段摘要，只有某個位置之後的訊息原樣保留。

### 安全性（p.82）
Pi 以和自己行程相同的權限執行，也不會跳出視窗請你確認，所以要**當成永遠在自動模式、沒有任何保護**。最好在容器或沙盒裡跑。

### 用本講的模式對照 Pi

| 決策 | Pi 的選擇 | 對應的模式 |
|---|---|---|
| 推理與行動 | ReAct＋JSON 工具呼叫 | ReAct（3.1）；不是 CodeAct，但有 bash 所以隨時能寫程式執行，某種程度兼得兩者 |
| 規劃 | 不內建 | 放棄 ReWOO 式先規劃，交給使用者或擴充 |
| 上下文 | 事件日誌＋skill＋壓縮 | 4.4、4.7、4.6 |
| 子 agent | 不做 | 放棄上下文隔離的「子 agent」那一半 |
| 安全 | 沒有權限檢查 | — |

> 💭 **對照我常用的 Claude Code**：它內建了 Pi 刻意不做的東西：plan mode（先規劃再執行）、子 agent（上下文隔離）、待辦清單（記憶卸載的一種）、權限確認。兩者是光譜的兩端：Pi 把決定權留給使用者，Claude Code 把常用模式先做好。

---

## 6. 自我檢查（p.84–86）

**Q1. Workflow 和 agent 的差別是什麼？STORM 用到了五種 workflow 模式中的哪幾種，各在哪個環節？**（p.9–12、p.24）
- 差別在**控制流程由誰決定**。Workflow 的流程寫在程式裡（提示鏈、路由、分支都是我們事先寫好的），好預測、好除錯；agent 由 LLM 依觀察結果決定下一步，比較彈性。
- STORM 用到四種：
  - 調查 → 大綱 → 精修：**提示鏈**
  - 多個視角各自對話：**平行化**
  - 依視角分派對話：**協調者–工作者**（其實接近平行化）
  - 作者與專家一問一答：大致是**評估者–優化者**

**Q2. 純粹讓 LLM 檢查自己的答案常常沒用，為什麼 Reflexion 和 CRITIC 卻有效？**（p.38–42）
- 自我修正沒有新資訊進來，模型只能拿產生錯誤的同一套知識檢查，常把對的改錯（p.41）。
- Reflexion 的評估來自**外部環境**（測試結果、環境獎勵）；CRITIC 讓驗證者能**用外部工具**查證（搜尋、程式直譯器）。
- 兩者都滿足了評估者–優化者的條件（p.20）：回饋清楚，而且有東西能給出可靠的回饋。
- 另外 Reflexion 把反思**存成記憶**帶到下一輪，比只看上一輪軌跡更有效（0.66 → 0.74）。

**Q3. 觀察遮蔽和壓縮各怎麼做？為什麼觀察遮蔽特別適合 coding agent？**（p.65–67）
- **壓縮**：用 LLM 把舊的推理、行動、觀察摘要成一段文字。
- **觀察遮蔽**：不呼叫 LLM，只把舊的工具輸出換成佔位符或截短，推理和行動保留原樣。
- 適合 coding agent 的原因：觀察佔了 83.9% 的 token；檔案內容和測試輸出很長、很快過時，而且需要時可以再讀一次。遮掉它省最多 token，又保留了「做過什麼、為什麼」。成本更低，效果和摘要差不多甚至更好。

---

## 待辦
- [ ] 讀 ReAct 論文 §2（方法）、§3（知識型任務與錯誤分析）、§4（決策型任務）
- [ ] 選讀：Building Effective Agents、Effective Context Engineering
- [ ] 預習 Lecture 7：MemGPT
