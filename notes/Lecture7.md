# Lecture 7：Agent Memory Architectures Agent 記憶架構

- 日期：10/14（第 4 週）
- 投影片：[lecture07.pdf](../slides/lecture07.pdf)
- 必讀：[MemGPT](http://127.0.0.1:8329/reading/Packer2023_MemGPT.html)

## 閱讀清單

| 類別 | 文章 | 本站譯文 | 一句話 |
|---|---|---|---|
| **必讀** | [MemGPT: Towards LLMs as Operating Systems](http://127.0.0.1:8329/reading/Packer2023_MemGPT.html) | 全譯對照 | 借用作業系統虛擬記憶體與分層儲存的概念，替上下文固定的 LLM 建立能自我管理的記憶層級，用於長對話和長文件分析。 |
| 推薦 | [Agent Memory: How to Build Agents That Learn and Remember](http://127.0.0.1:8329/reading/Letta2025_Agent_Memory_blog.html) | 全文中譯 | Letta 團隊（MemGPT 作者）整理 agent 記憶的分類（訊息緩衝、核心／回憶／歸檔記憶）與建構技術，提出「記憶就是上下文工程」的觀點。篇幅短，建議優先讀。 |
| 推薦 | [Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory](http://127.0.0.1:8329/reading/Chhikara2025_Mem0.html) | 全譯對照 | 提出 Mem0 與圖記憶變體 Mem0g：從對話中動態抽取、整合、檢索記憶，在 LOCOMO 上領先，並大幅降低延遲與 token 用量。 |
| 推薦 | [Generative Agents: Interactive Simulacra of Human Behavior](http://127.0.0.1:8329/reading/Park2023_Generative_Agents_UIST2023.html) | 全譯對照 | 史丹佛小鎮論文：用記憶流、檢索、反思、規劃驅動 25 個 LLM agent 模擬可信的人類行為，並出現資訊擴散、協作等社會現象。很長，可以只讀記憶與反思的章節。 |

> 必讀一定要讀；推薦、補充可以依時間和興趣挑。清單依本站 index.html 的閱讀材料整理。

---

這份筆記把 Lecture 6 的 ReAct 和本講的 MemGPT 放在一起看，說明兩篇論文怎麼落地成一個系統。ReAct 的論文重點和迴圈實作見 [Lecture 6 筆記](Lecture6.md)。

兩篇論文都在證明「這個方法有效」並說明核心機制，不會把正式產品要的後端、資料庫、權限、監控寫出來。落地時，不是把論文「裝進 LLM」，而是在 LLM 外面寫一個 **agent controller**（也就是 [Lecture 5 筆記](Lecture5.md)第 4 節說的 harness），由它串起模型、工具和記憶。

> 這份筆記的論文數字都已對照 repo 裡的譯文原文查證過；標「工程建議」的部分是實作經驗，論文沒有寫。

## 1. 先用一句話抓住兩篇

| | ReAct（Yao et al., ICLR 2023） | MemGPT（Packer et al., 2023） |
|---|---|---|
| 解決什麼 | 只推理（CoT）會**幻覺**；只行動（Act）不會**規劃** | 上下文視窗**有限**，長對話、長文件塞不下 |
| 核心想法 | 讓模型**交錯**產生 Thought（推理）與 Action（行動），環境回傳 Observation | 借作業系統的**虛擬記憶體**：上下文是「記憶體」，外部儲存是「硬碟」，由 LLM 自己用函式呼叫換進換出 |
| 對應系統哪一塊 | 決策迴圈（第 3–6 步） | 記憶管理（第 1、7 步，以及迴圈中的記憶工具） |
| 關鍵數字 | ALFWorld、WebShop 只用 1–2 個示例，成功率分別比模仿／強化學習方法**絕對**高 34%、10% | 深度記憶檢索（DMR）上，GPT-4 從 32.1% 提升到 **92.5%** |

## 2. 實際系統長這樣

```
Vue 3 前端
   │
   ▼
FastAPI 後端 / Agent API（agent controller）
   │
   ├── 1. 搜尋長期記憶（MemGPT 的外部儲存，或 Mem0）
   ├── 2. 組合 prompt：系統指令 ＋ 相關記憶 ＋ 使用者問題 ＋ 可用工具清單
   ├── 3. 呼叫 LLM
   ├── 4. LLM 決定是否使用工具（搜尋文章、查 PostgreSQL、讀文件、呼叫企業 API）
   ├── 5. 後端執行工具，把結果交回 LLM
   ├── 6. 重複 3–5，直到產生答案
   └── 7. 把值得保留的新資訊寫入長期記憶
```

- **ReAct** 負責第 3–6 步：判斷、呼叫工具、看結果、再判斷。
- **記憶層** 負責第 1、7 步。MemGPT 和 Mem0 做法不同（見第 5 節）：MemGPT 讓 LLM 在第 3–6 步的迴圈裡**自己**呼叫記憶函式；Mem0 是迴圈**外面**的前處理和後處理。

---

## 3. MemGPT：讓 LLM 自己管理記憶（必讀）

MemGPT 把上下文視窗看成作業系統的**記憶體**、外部資料庫看成**硬碟**。LLM 透過函式呼叫自己決定什麼該留在上下文、什麼該移出去、什麼時候去查。這套機制叫**虛擬上下文管理**（virtual context management）。

**記憶分層**

| 層 | 位置 | 內容 | 誰能改 |
|---|---|---|---|
| 系統指令 System instructions | 主上下文（prompt 內） | MemGPT 的控制流程、各層記憶怎麼用、函式說明 | 唯讀 |
| 工作上下文 Working context | 主上下文 | 固定大小的文字區，放使用者的關鍵事實、偏好、agent 的人設 | 只能透過函式改 |
| FIFO 佇列 | 主上下文 | 最近的訊息、系統訊息、函式的輸入輸出；**第一格是被移出訊息的遞迴摘要** | 由佇列管理器管理 |
| 回憶儲存 Recall storage | 外部（資料庫） | **完整**的對話紀錄，被移出上下文的訊息也都在這裡 | 自動寫入，用函式查 |
| 歸檔儲存 Archival storage | 外部（資料庫） | 任意長度的文字，例如文件、長期筆記；論文實驗用 PostgreSQL + pgvector | 用函式寫入、查詢 |

論文中出現的記憶函式：`working_context.append`、`working_context.replace`、`recall_storage.search`、`archival_storage.search`。

**上下文快滿時怎麼辦（佇列管理器）**

```
prompt 超過「警告門檻」（例如上下文的 70%）
  → 插入一則系統訊息：「記憶壓力」警告
  → LLM 自己把重要資訊存進 working context 或 archival storage
prompt 超過「清空門檻」（例如 100%）
  → 移出一部分訊息（例如上下文的 50%）
  → 用「舊摘要 ＋ 被移出的訊息」產生新的遞迴摘要，放在佇列第一格
  → 被移出的訊息仍永久存在 recall storage，之後可以用函式查回來
```

**控制流程的兩個機制**

- **事件觸發**：使用者訊息、系統警告、使用者登入或上傳完文件的通知、**定時事件**都會觸發一次推論。所以 MemGPT 可以在沒人說話時自己運作（Lecture 5 的 OpenClaw 每 30 分鐘 heartbeat 就是這個概念）。
- **函式串接**（function chaining）：呼叫函式時帶上 `request_heartbeat=true`，函式結果放回上下文後**立刻再跑一次 LLM**，可以連續查好幾頁搜尋結果；沒帶這個旗標就是 **yield**，暫停直到下一個外部事件。

**實驗**

- **對話**：在 Multi-Session Chat 上設計「深度記憶檢索」（DMR），問題只能靠前幾次對話的內容才答得出來。基準模型只看得到前幾次對話的有損摘要；MemGPT 可以分頁搜尋完整歷史。GPT-4 從 32.1% 提升到 92.5%，GPT-4 Turbo 從 35.3% 到 93.4%，GPT-3.5 從 38.7% 到 66.9%。
- **文件分析**：固定上下文的基準模型受限於檢索器一次給的文件；MemGPT 可以**多次**查 archival storage。在巢狀鍵值檢索任務中，GPT-4 到第 3 層巢狀就降到 0%，MemGPT（GPT-4）不受巢狀層數影響。

**放進 ReAct 迴圈的樣子**：記憶函式就是**一般的工具**，跟 `search_articles` 一起註冊。差別只在 controller 多了兩件事：組 prompt 時放進 working context 和遞迴摘要；每輪檢查 token 數，必要時插入警告或清空佇列。

```python
MEMORY_TOOLS = {
    "working_context_append": ...,   # 記下使用者的關鍵事實
    "working_context_replace": ...,  # 更正過時的事實（例如 Vuex → Pinia）
    "recall_storage_search": ...,    # 查以前的對話
    "archival_storage_insert": ...,  # 存長期資料
    "archival_storage_search": ...,  # 查長期資料
}

def build_messages(state):
    return [
        {"role": "system", "content": SYSTEM_INSTRUCTIONS},
        {"role": "system", "content": f"工作上下文：\n{state.working_context}"},
        {"role": "system", "content": f"先前對話摘要：\n{state.recursive_summary}"},
        *state.fifo_queue,
    ]

def manage_context(state):
    tokens = count_tokens(build_messages(state))
    if tokens > 0.7 * CONTEXT_LIMIT:
        state.fifo_queue.append({"role": "system", "content": "記憶壓力警告：請把重要資訊存起來"})
    if tokens > CONTEXT_LIMIT:
        evicted = pop_oldest(state.fifo_queue, ratio=0.5)          # 舊訊息本來就已寫進 recall storage
        state.recursive_summary = summarize(state.recursive_summary, evicted)
```

（概念示意，不是 MemGPT 原始碼。MemGPT 後來演變成 Letta，產品化的版本見推薦閱讀的 Letta 部落格。）

---

## 4. Mem0：在迴圈外面做記憶（推薦）

Mem0 不讓主 agent 自己管記憶，而是在**回答前**搜尋、**回答後**抽取和更新，是一條獨立的記憶管線。

**回答前：讀取記憶**

```python
memories = memory.search(
    query=user_message,
    filters={"user_id": user_id},   # 新版 SDK：user_id 要放在 filters 裡
)
```

問題會轉成 embedding，在該使用者的記憶範圍內做向量搜尋，可以再用篩選條件或 reranker 排序。官方文件建議至少用 `user_id` 限定範圍，避免不同使用者的記憶互相污染。

> **API 版本注意**：官方文件現在把 `user_id`、`agent_id` 放在 `filters` 裡，而且直接寫成 `search(query, user_id=...)` 會報錯；但一些舊文件和範例仍是這種舊寫法。照著範例寫卻出錯時，先確認 SDK 版本。

例如使用者問「請依照我們專案規範建立 Pinia Store」，可能找回：

```json
[
  { "memory": "此專案使用 Vue 3、Vite 8 與 Pinia。" },
  { "memory": "所有 Store 寫入必須透過 actions。" },
  { "memory": "元件解構 Store 時使用 storeToRefs。" }
]
```

後端再把它放進 prompt：

```
系統指令：你是一位前端工程助理。
相關長期記憶：
- 此專案使用 Vue 3、Vite 8 與 Pinia。
- 所有 Store 寫入必須透過 actions。
- 元件解構 Store 時使用 storeToRefs。
使用者問題：請建立使用者權限 Store。
```

不是 LLM 自己突然想起來，而是**後端先搜尋記憶，再放回 prompt**。

**回答後：寫入記憶**

```python
memory.add(
    messages=[
        {"role": "user", "content": user_message},
        {"role": "assistant", "content": assistant_answer},
    ],
    user_id=user_id,
)
```

論文裡 `add` 背後是**兩個階段**：

| 階段 | 做什麼 | 論文設定 |
|---|---|---|
| ① 抽取 Extraction | 拿「整段對話的摘要 ＋ 最近 m 則訊息 ＋ 這一輪的問答」，讓 LLM 抽出候選事實。不重要的內容在這一步就不會被抽出來 | m = 10 |
| ② 更新 Update | 每個候選事實先找出 s 筆最相似的既有記憶，再讓 LLM 用 tool call 從四種操作中選一個 | s = 10，模型用 GPT-4o-mini |

四種操作：

| 操作 | 時機 | 例子 |
|---|---|---|
| `ADD` | 沒有相似的既有記憶 | 新增「Store 寫入必須透過 actions」 |
| `UPDATE` | 有相關記憶，新事實補充了更多資訊（「合併」也是用這個操作） | 「使用 Pinia」→「使用 Pinia，元件解構用 storeToRefs」 |
| `DELETE` | 新事實和舊記憶矛盾 | 使用者說「已完成 Pinia 遷移」→ 刪除「專案仍使用 Vuex」 |
| `NOOP` | 已經記過了，或不需要改 | 再說一次「我們用 Pinia」 |

> 修正：原筆記列了五種判斷（新增、更新、刪除、合併、不值得保存）。論文只有上面四種：「合併」屬於 UPDATE；「不值得保存」在抽取階段就被過濾掉，或在更新階段判為 NOOP。

這才是 Mem0 和「單純保存聊天紀錄」的差別。論文另外提出圖記憶變體 Mem0g，在 LOCOMO 上和全上下文基準相比，p95 延遲降低 91% 以上。

## 5. MemGPT 和 Mem0 怎麼選

| | MemGPT | Mem0 |
|---|---|---|
| 誰決定存什麼、查什麼 | **主 agent 自己**，透過記憶函式 | **獨立的記憶管線**，在回答前後自動執行 |
| 放在系統哪裡 | ReAct 迴圈**裡面**（記憶就是工具） | ReAct 迴圈**外面**（前處理、後處理） |
| 上下文管理 | 有：記憶壓力警告、遞迴摘要、佇列清空 | 不管上下文，只管長期記憶 |
| 優點 | 能處理超長文件、多次分頁查詢；agent 可以臨場決定要不要查 | 主 agent 簡單；行為可預期，容易加到現有系統 |
| 代價 | 每次記憶操作都佔推論步數；要靠模型自己記得去存 | 每輪都要多跑抽取和更新的 LLM 呼叫；查什麼由相似度決定，不是由 agent 決定 |

用 Lecture 5 的光譜來看：Mem0 的記憶流程是**寫死的 workflow**，MemGPT 把記憶的控制權**交給 agent**。兩者不衝突，也可以混用：用 Mem0 做每輪自動記憶，同時把 `archival_storage_search` 當成工具給 agent 臨時查資料。

---

## 6. 把 ReAct 和記憶接起來（以 Mem0 為例）

```python
def run_agent(user_id: str, user_message: str) -> str:
    # 1. 取得相關長期記憶
    memories = memory.search(query=user_message, filters={"user_id": user_id})

    # 2. 建立本次上下文
    messages = [
        {"role": "system", "content": build_system_prompt(memories)},
        {"role": "user", "content": user_message},
    ]

    # 3. ReAct / tool-calling 迴圈（Lecture 6 筆記第 2 節的 react_loop）
    final_answer = react_loop(messages, max_steps=8)

    # 4. 儲存本輪值得保留的資訊（只在正常完成時寫入）
    if final_answer != "超過最大步數，停止":
        memory.add(
            messages=[
                {"role": "user", "content": user_message},
                {"role": "assistant", "content": final_answer},
            ],
            user_id=user_id,
        )
    return final_answer
```

> 修正：原版在 `for` 迴圈跑滿 8 步都沒有 `break` 時，`final_answer` 從未被賦值，執行到 `memory.add` 會丟出 `NameError`。改成重用 `react_loop`，它一定有回傳值。

**流程**：① 使用者提問 → ② 搜尋長期記憶 → ③ 記憶放入上下文 → ④ LLM 判斷是否需要工具 → ⑤ 後端執行工具 → ⑥ 結果送回 LLM → ⑦ 重複 ④–⑥ 直到完成 → ⑧ 回答使用者 → ⑨ 抽取並保存新的重要資訊。

## 7. 用我熟悉的技術怎麼組（工程建議）

| 層級 | 技術 | 職責 |
|---|---|---|
| 前端 | Vue 3、Pinia | 對話介面、顯示工具執行狀態 |
| Agent API | FastAPI | 接收問題，用串流回傳結果 |
| Agent 編排 | LangGraph 或自己寫迴圈 | 管理工具呼叫、狀態、重試與停止 |
| LLM Gateway | LiteLLM | 統一串接不同模型，記錄 token 與成本 |
| 長期記憶 | Mem0（或 Letta） | 記憶抽取、更新、搜尋 |
| 業務資料 | PostgreSQL | 文章、使用者、任務、操作紀錄 |
| 快取／佇列 | Redis | 工作狀態、暫存、背景任務 |
| 向量儲存 | Qdrant、pgvector | 儲存與搜尋記憶的 embedding |
| 工具 | Python 函式／內部 API | 搜尋文章、讀檔、查資料庫 |

**第一版 PoC** 不用一次導入全部：Vue 3 ＋ FastAPI ＋ LiteLLM ＋ Mem0 OSS ＋ PostgreSQL/pgvector ＋ 3 個**唯讀**工具（`search_articles`、`get_article`、`search_project_docs`）。先不要讓 agent 修改資料或執行任意 shell 指令。

## 8. 正式環境真正困難的地方（工程建議，論文沒處理）

| 問題 | 做法 |
|---|---|
| **權限隔離** | 記憶搜尋一律帶範圍。單一產品用 `user_id` 可能就夠；企業環境再加 `tenant_id`（或放在 metadata 篩選），避免跨組織資料外洩 |
| **工具白名單** | 模型只能呼叫事先註冊的工具（[Lecture 6 筆記](Lecture6.md)第 2 節的 `execute_tool` 已經做了） |
| **寫入操作確認** | 唯讀工具（搜尋文件、查資料庫、讀文章）可以自動執行；有副作用的工具（改資料、寄信、建 Jira、部署）要人工確認，不能在 ReAct 迴圈裡直接執行。HW1 的 interactive（confirm/auto 模式）就是在練這個 |
| **停止條件** | 最大工具呼叫次數（例如 8 次，對照論文 HotpotQA 用 7 步）、單次工具逾時 10–30 秒、整體任務逾時 60–120 秒、token 上限依預算設定 |
| **記憶治理** | 帳號密碼、token、身分證字號、醫療資訊、短期臨時內容不寫入長期記憶；使用者要能查看、更正、刪除自己的記憶 |

## 最關鍵的理解

論文提供設計原理，工程框架與後端程式才把原理變成產品。

| 元件 | 負責 |
|---|---|
| LLM | 決定下一步、產生答案 |
| ReAct controller | 反覆呼叫 LLM 和工具，直到完成 |
| Tool registry | 定義 agent 可以做哪些事 |
| 記憶層（MemGPT／Mem0） | 搜尋、建立、更新長期記憶；MemGPT 還管上下文 |
| Database | 真正保存資料 |
| Backend | 權限、流程、錯誤處理、安全 |

做一個 agent 不是呼叫一次 LLM API，而是寫出「**查記憶 → 呼叫模型 → 執行工具 → 回傳觀察結果 
→ 再呼叫模型 → 保存記憶**」這條控制流程。這也正是 HW1 要你親手做的事。


---

## PDF 內容筆記（整理版）

> 逐頁的完整整理見 [Lecture 7 導讀](Lecture7-guide.html)。這裡只留重點，加上我讀的時候的疑問與解答。

### 1. 記憶類型（p.15–22）

CoALA 把 agent 的記憶分成四類：

| 類型 | 存什麼 | 例子 |
|---|---|---|
| 工作記憶 Working | 當下看得到、正在推理的內容，通常就是**上下文視窗** | 這一輪的觀察、目標、使用者輸入 |
| 情節記憶 Episodic | 過去行為的**原始紀錄**（軌跡、對話） | Generative Agents 的記憶流 |
| 語意記憶 Semantic | 關於世界與使用者的**事實** | ChatGPT「管理記憶」裡的條目 |
| 程序記憶 Procedural | **怎麼做事**的規則與技能 | Claude 的 Skill、Voyager 的技能庫 |

另一個切法是記憶放在哪裡：

| | 參數記憶（在模型權重裡） | 上下文記憶（放進模型上下文） |
|---|---|---|
| 語意 | 預訓練學到的知識 | 檢索來的事實 |
| 程序 | 指令遵循能力 | 檢索來的技能 |
| 情節 | **不適用** | 檢索來的過去對話 |
| 怎麼更新 | 微調 | 由上下文管理器挑出合適的記憶放進來 |

情節記憶在參數裡「不適用」，因為權重記不住某一次具體發生的事。本講的架構幾乎都在做**上下文記憶**。

### 2. 設計記憶的三個問題與三種儲存（p.24–28）

- **What**：存什麼？事實、程序、對話進展。
- **When**：什麼時候存取？agent 用工具隨需呼叫、背景程序、事件觸發。
- **How**：怎麼存？

| 儲存 | 做法 | 適合 |
|---|---|---|
| 檔案系統 | 每筆記憶一個檔案，用 `grep`、`ls` 當記憶工具；資料夾天生有階層 | coding agent（`AGENTS.md`、`CLAUDE.md`、skill 資料夾） |
| 資料庫 | 向量資料庫做語意搜尋；SQLite 依中繼資料（時間、使用者、類型）篩選；BM25 做關鍵字精確比對 | 大量、零散的事實與對話 |
| 知識圖譜 | 實體—關係—實體，關係可帶時效（Zep） | 事實會隨時間改變、需要多跳查詢 |

### 3. 四種架構模式（p.29–66）

| 模式 | 代表 | 一句話 | 主要取捨（p.66） |
|---|---|---|---|
| ① 從經驗學程序 | Voyager、AWM、Skill 檔 | 把成功的做法存成可重用的程式、workflow 或 skill | 省力，但環境一變程序就過時 |
| ② 分層記憶 | MemGPT、MemoryOS | 上下文放精簡資訊，細節放外部，需要時換進來 | 上下文變小，就要靠可靠的記憶管理 |
| ③ 反思式整合 | Reflexion、Generative Agents | 先反思、提煉成教訓或推論再存 | 抽象化利於重用，但可能產生沒根據的結論 |
| ④ 記憶維護 | 睡眠時計算、A-Mem、Mem0 | 在背景整理、合併、刪除舊記憶 | 要多花計算，也可能丟失資訊 |

**模式一重點**

- Voyager：技能的**說明的嵌入向量當 key、程式本身當 value** 存進技能庫；新任務時用任務說明的嵌入取回最相關的 5 個技能。有技能庫才做得出鑽石工具。
- AWM：從成功軌跡歸納出 workflow，放進**系統提示**當解題的上下文。

> **Q：key／value 的說法，就是 skill 的意思嗎？**
>
> 是同一個概念（程序記憶）的兩種實作，但存取方式不同：
>
> | | Voyager 技能庫 | Claude Skill |
> |---|---|---|
> | 內容 | 一段可執行的 JavaScript 程式 | 一個資料夾：`SKILL.md`＋參考檔＋腳本 |
> | 誰寫 | agent 自己寫，通過自我驗證才存 | 主要由人寫（也可以讓 agent 產生） |
> | 怎麼找 | 用**嵌入相似度**自動取回 top-5 | 系統提示列出每個 skill 的 `name`／`description`，**模型讀了自己決定**要不要打開 |
>
> 對應起來：`description` ≈ key，`SKILL.md` 內容 ≈ value。差別在 Voyager 靠向量檢索，Skill 靠模型讀說明後判斷（漸進揭露）。

**模式二重點**

- MemGPT 上下文切成：系統指令（唯讀）、工作上下文（用 `append`／`replace` 函式改）、FIFO 佇列、輸出緩衝。上下文外有回憶儲存（完整對話）與歸檔儲存（長期資料）。
- MemoryOS 用短、中、長期三層，換層靠**規則**（FIFO、熱度門檻），不是讓 LLM 自己決定。

**模式三重點**

- 不直接存原始經驗，而是先反思、提煉後再存。Reflexion 在 HotPotQA 上：只用 CoT 約 0.61 → 加情節記憶約 0.66 → 再加反思約 0.74。

> **Q：讓同一個 LLM 反思自己，不是容易出問題嗎？那要怎麼「先反思」？**
>
> 你的直覺是對的：Lecture 6 講過，**沒有外部回饋**時讓 LLM 檢查自己的答案，常常沒有改善，甚至改錯。兩篇論文之所以有效，是因為反思**有依據**：
>
> - **Reflexion**：反思之前先有**評估者**給的外部訊號（單元測試結果、環境回報成功或失敗）。模型不是憑空判斷對錯，而是在「已知失敗了」的前提下分析原因。
> - **Generative Agents**：反思不是檢查對錯，而是**歸納**。最近事件的重要性分數加總超過門檻（論文設 150，實際約一天 2–3 次）時，拿最近 100 筆記憶問「最值得注意的 3 個問題」，再生成洞見，而且每則洞見都要**引用它根據的記憶**。
>
> 實作上可以這樣降低風險：
> 1. 反思前要有外部訊號（測試、使用者回饋、任務成敗），沒有就不寫反思。
> 2. 反思要附上來源（引用哪幾筆原始記憶），方便查證與回溯。
> 3. **原始紀錄要保留**，反思只是加一層摘要，不取代原始資料；反思錯了可以重新產生。
> 4. 把反思標成「推論」，信心比直接觀察到的事實低；和事實衝突時以事實為準。

**模式四重點**

- 睡眠時計算：沒人提問時先整理上下文（例：把四條零散的顏色偏好整理成一份排序清單）。
- A-Mem：建一個**專門管理記憶的 agent**。四步驟：建筆記（時間戳、內容、情境、關鍵字、標籤、嵌入）→ 產生連結（找 top-k 相近筆記，由 LLM 判斷要不要連）→ 記憶演化（新筆記進來時，決定是否更新舊筆記）→ 檢索。
- Mem0：抽取階段抽出候選事實，更新階段由 LLM 從 `ADD`／`UPDATE`／`DELETE`／`NOOP` 中選一個，在**背景**自動處理衝突。

> **Q：「專門管理記憶的 agent」是常見做法嗎？這和 prompt cache 一樣嗎？**
>
> **常見，但多半不叫「agent」，而是「背景記憶管線」**：主 agent 負責回答，另一條 LLM 流程在回答後（或閒置時）抽取、合併、刪除記憶。Mem0、Google 的 Vertex AI Memory Bank、LangMem 的背景模式、Letta 的 sleep-time agent 都是這個形狀（見下一節查證）。像 A-Mem 那樣讓它自主決定連結與演化的比較少見，多數產品用固定的抽取→更新流程，因為比較好預測與除錯。
>
> **prompt cache 是另一件事**，兩者不要混在一起：
>
> | | 記憶（memory） | 提示快取（prompt caching） |
> |---|---|---|
> | 目的 | 讓 agent **知道**以前發生的事 | 讓同樣的內容**算得更快、更便宜** |
> | 改變上下文內容嗎 | 會，決定放什麼進上下文 | 不會，只是重用已經算過的前綴 |
> | 跨對話嗎 | 會，存到資料庫或檔案 | 不會，快取幾分鐘就過期，而且前綴要一字不差 |
>
> 兩者的關係是：把穩定的記憶（使用者檔案、專案規範）放在 prompt **前段**，前綴才不會常變，快取才命中得了（Lecture 6 講 KV cache 時提過）。

> **Q：長期下來記憶越積越多，A-Mem 這類做法要怎麼落地比較合適？**
>
> 重點是控制**寫入**與**定期整理**，而不是全部存下來再想辦法找：
>
> 1. **限定範圍**：每筆記憶都帶 `user_id`（企業再加 `tenant_id`、`project_id`），搜尋時一律帶上。
> 2. **寫入前過濾**：只存長期有用的事實、偏好、決定；閒聊、一次性的內容不存（Mem0 在抽取階段就濾掉）。
> 3. **寫入時去重**：先找相似的舊記憶，再決定新增、更新、刪除或不動，不要每次都新增一筆。
> 4. **保留時間資訊**：事實改變時，記下「何時起有效、何時失效」，而不是直接覆蓋（Zep 的做法）；才答得出「以前喜歡什麼、為什麼改了」。
> 5. **背景整理**：定期（例如每晚）合併重複、摘要舊記憶，像睡眠時計算。
> 6. **淘汰機制**：依最近使用時間與被取用次數算熱度，冷掉的移到冷儲存或刪除（MemoryOS 的做法）。
> 7. **檢索有上限**：每次只取 top-k（例如 5–10 筆），避免把整個記憶庫塞進上下文。
> 8. **使用者可控**：能查看、更正、刪除自己的記憶。
>
> 第一版可以先只做 1、2、3、7；記憶量真的大了，再加 4、5、6。

### 4. 評估 agent 記憶（p.68–94）

好的記憶基準要滿足四個條件：**真實**（記憶來源和查詢要反映真實情境）、**長上下文**（短對話用不著記憶）、**有挑戰性**（太簡單分不出差異）、**可驗證**（要能判斷成敗）。

| 基準 | 記憶來源 | 怎麼測 | 不足（🟨） | 重要發現 |
|---|---|---|---|---|
| LoCoMo | 兩個模擬使用者的長對話（約 16.6k token） | 問答、事件摘要、多模態對話 | 真實、長上下文 | 就算取回對的記憶，模型也不一定用得好 |
| PersonaMem | 使用者與助理的多 session 對話，偏好會變（最長近 1M token） | 選擇題 | 真實、有挑戰性 | 記憶／RAG 是必要的；Mem0 有助追蹤偏好變化 |
| LongMemEval-v2 | 網頁 agent 的歷史軌跡（25M–115M token） | 451 題，多數是簡答 | 真實 | **檔案系統式記憶勝過檢索式記憶**（但那組用的模型也比較強） |
| MemoryArena | 多 session、子任務互相依賴的真實任務 | 實際完成任務 | 無，四項都通過 | **記憶 agent 是最弱的方法**，進步空間很大 |

### 5. 挑戰與開放問題（p.96–100）

- **記什麼、忘什麼**：建立記憶時不知道未來是否有用。全存，成本變高；忘掉，可能丟失證據或還沒兌現的承諾。例：「10 點前的會議我不承認，除非有糕點」——是偏好還是玩笑？
- **記憶下毒**（AgentPoison）：攻擊者在記憶或 RAG 知識庫植入少量惡意示範，並最佳化一個觸發詞。指令帶有觸發詞（例如「開得平順安全」）時，就會取回惡意示範，讓自駕 agent 突然全力煞車。
- **隱私**（PrivacyLens）：agent 帶著記憶與外界互動，可能洩漏你不想公開的事。例：寫給主管的週報裡出現「和其他公司的招募人員午餐」。
- **過度個人化**：模型太急著用上下文裡的記憶。例：問「年輕選民的角色」，卻回「就像你在生活中做健康的小改變……」。

### 6. 自我檢查（附答案）

**① 四種記憶各存什麼？哪一種沒辦法存在模型參數裡？**
工作記憶是當下的上下文；情節記憶是過去行為的原始紀錄；語意記憶是事實；程序記憶是技能與規則。**情節記憶**沒辦法存在參數裡，因為權重學到的是統計規律，記不住「某一天發生了某件事」這種單一事件。

> 原本寫的「推理迴圈、程序記憶透過 Prompt／Parse 交換資料」是 CoALA 架構圖裡**工作記憶**的描述，不是這題的答案。

**② MemGPT 的上下文切成哪幾塊？它和 Mem0 在「誰負責更新記憶」上有什麼不同？**
系統指令（唯讀）、工作上下文（透過函式讀寫）、FIFO 佇列（由佇列管理器寫入）、輸出緩衝。MemGPT 由**主 agent 自己**在推理迴圈裡呼叫 `append`／`replace` 更新記憶；Mem0 由迴圈**外面的背景管線**在回答後自動抽取與更新，主 agent 不用管。

**③ 好的記憶基準要滿足哪四個條件？LoCoMo 哪兩項不夠好？MemoryArena 的結果代表什麼？**
真實、長上下文、有挑戰性、可驗證。LoCoMo 在**真實**（模擬的是使用者之間的對話，不是使用者與 agent）和**長上下文**（每段只有約 16.6k token）兩項不夠好。MemoryArena 中記憶 agent 的平均成功率（0.12–0.14）比單純的 RAG（0.17–0.20）還低，代表現有記憶系統在真實、多步驟任務上還不可靠，不過任務本身很難，失敗不能全怪記憶。

---

## 延伸查證：現在主流是用 MemGPT 嗎？（2026-11 查證）

### 結論

**不是。** MemGPT 的概念影響很大，但現在沒有單一主流框架，而是幾種做法並存：

| 做法 | 代表 | 怎麼運作 | 誰在用 |
|---|---|---|---|
| **背景抽取＋回合開始時載入**（Mem0 型） | Mem0、Vertex AI Memory Bank、LangMem | 回答後由 LLM 抽取事實、和舊記憶合併並解決矛盾；下一輪開始前把相關記憶放進系統提示 | 在既有聊天產品或 agent 上「加記憶」最常見的做法 |
| **檔案式記憶** | `CLAUDE.md`／`AGENTS.md`、Skill、Claude API 的 memory tool | 記憶就是檔案，agent 用讀寫檔案的工具管理 | coding agent 幾乎都是這種；本講 p.85 也顯示它在 LongMemEval-v2 勝過檢索 |
| **agent 自己管理分層記憶**（MemGPT 型） | Letta | 記憶區塊常駐上下文，agent 用工具自己改；外部有對話歷史與歸檔儲存 | 需要長期運作、有狀態的單一 agent |
| **時序知識圖譜** | Zep（Graphiti） | 把事實存成帶時效的實體關係 | 事實常變、需要追蹤「以前 vs 現在」的情境 |

幾個值得注意的趨勢：

- **雲端大廠都推出了託管記憶服務**，而且多半是 Mem0 型。例如 Google 的 Vertex AI Memory Bank 用 Gemini 從對話中產生長期記憶，並在新資訊進來時合併、解決矛盾；搭配 Google ADK 的 `PreloadMemoryTool`，每一輪開始時把記憶放進系統指令。文件同時也提供「記憶當工具」的做法，讓 agent 自己決定要寫什麼，也就是 MemGPT 的思路。
- **Anthropic 的 memory tool 是檔案式的**：Claude 用 view、create、str_replace 等指令操作 `/memories` 目錄，儲存後端由開發者自己實作（本機檔案、資料庫、雲端都可以）；另外搭配 context editing，在接近 token 上限時自動清掉舊的工具結果。
- **常見的是混用**，不是選一個：例如偏好用 Mem0 型自動記，專案規範放檔案，對話歷史放資料庫。

### 我的理解：MemGPT 這個「框架」不是主流，但它的「零件」到處都是

MemGPT 論文提出的每個機制，現在幾乎都能在主流工具裡找到，只是被拆開來、各自實作：

| MemGPT 的機制 | 現在常見的樣子 |
|---|---|
| **工作上下文**：常駐在 prompt 裡、可以用函式修改的一小塊記憶 | 每次對話都會載入的使用者記憶或專案檔（`CLAUDE.md`、`AGENTS.md`）；Letta 的記憶區塊（memory blocks） |
| **記憶當工具**：agent 自己決定什麼時候存、存什麼 | Claude 的 memory tool、Vertex AI Memory Bank 的「記憶當工具」模式 |
| **記憶壓力警告＋遞迴摘要**：上下文快滿時先提醒、再移出並摘要舊訊息 | coding agent 的自動壓縮（compaction）；Claude 的 context editing 自動清掉舊的工具結果 |
| **回憶儲存**：完整對話歷史，需要時再搜尋 | 「搜尋過去對話」這類功能；把對話紀錄存進資料庫，再當成工具給 agent 查 |
| **歸檔儲存**：向量資料庫裡的長期資料 | 一般的 RAG、Mem0 的向量儲存 |
| **事件觸發**：沒人說話時也能被排程或通知喚醒 | 背景或排程 agent、Letta 的 sleep-time agent |

所以比較準確的說法是：**「讓 LLM 透過工具管理自己的上下文」這個想法已經成為共識，但很少人整套照搬 MemGPT 的架構。** 多數系統只取用自己需要的部分，例如只做壓縮、或只加一個記憶工具。

### 為什麼沒有一致的主流：本講的數據

本講的評估結果正好說明，目前還沒有一種做法明顯勝出：

- **檔案系統式記憶勝過檢索式記憶**（LongMemEval-v2，p.85）。這支持 coding agent 用檔案當記憶的做法；不過那組用的模型也比較強，不能完全歸功於記憶方式。
- **記憶 agent 是最弱的方法**（MemoryArena，p.91）。Letta、Mem0 等記憶 agent 的平均成功率 0.12–0.14，比單純 RAG 的 0.17–0.20 還低。讓 agent 自己管理記憶，在真實的多步驟任務上還不可靠。
- **取回對的記憶不等於用得好**（LoCoMo，p.73）。召回率 82.7% 時，答題 F1 只有 40.5。瓶頸不只在儲存和檢索，也在模型怎麼使用記憶。
- **每種模式都有代價**（p.66）。程序會過時、分層要靠可靠的管理、反思可能產生沒根據的結論、維護要多花計算。

結論：選記憶架構時，要看自己的痛點與限制，而不是找一個「最主流」的框架直接套用。

### 選擇時可以這樣判斷

```
你的問題是什麼？
├─ 檢索不準、查不到、要多步查詢
│    → 不是記憶問題：把檢索做成工具，讓 agent 在迴圈裡多次查（agentic RAG），並改善檢索本身
├─ 單次任務太長，上下文塞不下
│    → 上下文管理：壓縮、清掉舊的工具結果、交給子 agent（Lecture 6 的上下文模式）
├─ 跨對話要記得使用者的偏好與事實
│    → Mem0 型背景記憶（Mem0、Vertex AI Memory Bank、LangMem）
├─ 規範、流程、怎麼做事（程序記憶）
│    → 檔案式記憶：AGENTS.md／CLAUDE.md、Skill
├─ 事實常改變，要回答「以前 vs 現在」
│    → 時序知識圖譜（Zep）
└─ 要一個長期運作、會自己整理記憶的有狀態 agent
     → Letta（MemGPT 型）
```

### 對那段 AI 回答的查證

| 說法 | 判斷 | 說明 |
|---|---|---|
| 測試階段不用另外架資料庫，Letta 會自動用 SQLite | ✅ 大致正確 | 官方文件：用 pip 安裝時預設 SQLite，用 Docker 預設 PostgreSQL。但「存在 `~/.letta`」我沒查到 SQLite 的確切路徑；Docker 範例是把 PostgreSQL 資料放在 `~/.letta/.persist/pgdata`。另外，Letta 版本升級時**只正式支援 PostgreSQL 的資料遷移**，所以想長期保留的資料，一開始就用 PostgreSQL 比較保險 |
| 正式上線換成 PostgreSQL＋pgvector | ✅ 正確 | 官方文件：可用 `LETTA_PG_URI` 接自己的 PostgreSQL，資料庫必須裝 pgvector |
| 單一 agent 就能用，內建 Core／Recall／Archival 記憶 | ✅ 正確 | 這就是 MemGPT 論文的設計（工作上下文／回憶儲存／歸檔儲存），Letta 沿用 |
| 「主動、多輪迭代搜索」是 MemGPT 帶來的改善 | ⚠️ 歸因不精確 | 多輪搜尋、改寫關鍵字、翻頁，是 **agent 迴圈＋工具呼叫**（ReAct）的能力。任何能呼叫工具的 Gemini agent 都能做到，不需要 MemGPT。MemGPT 論文的函式串接（`request_heartbeat`）只是其中一種實現 |
| 糾正一次就「永遠記得」 | ⚠️ 過度宣稱 | 記不記得取決於模型有沒有呼叫寫入函式；寫錯、被覆蓋、記到不該記的都有可能（本講 p.96–100 的挑戰） |
| 「永遠不會爆 Context」 | ⚠️ 過度宣稱 | MemGPT 會在上下文快滿時移出舊訊息並遞迴摘要，所以不會超過上限；但摘要會丟資訊，不等於品質不會下降 |
| 多表關聯查詢（先查 A 表再查 B 表）適合用 MemGPT | ⚠️ 搞混了問題 | 這是**工具呼叫與推理**的問題（text-to-SQL、多步查詢），屬於 ReAct 的範圍。記憶能幫上的是記住資料表結構說明、你的查詢習慣與偏好 |
| 簡單的一問一答 FAQ 不需要 | ✅ 合理 | 沒有跨對話狀態的需求，就不必加記憶層 |

### 如果是「Gemini agent 做資料庫搜索」，我會這樣選

1. **先分清楚痛點**：
   - 「盲目檢索、抓錯資料、查不到」→ 把檢索做成**工具**、讓 agent 在迴圈裡多次查詢（agentic RAG），並改善檢索本身（混合檢索、reranker、更好的切塊）。這不需要記憶框架。
   - 「每次都忘記我的偏好、上次查過什麼」→ 這才需要長期記憶。
2. **需要長期記憶時**，已經在用 Gemini 的話，最省事的是 **Google ADK＋Vertex AI Memory Bank**；想自己掌控、不綁雲端就用 **Mem0**（可接 Gemini）。兩者都是在既有 agent 外面加一層，改動最小。
3. **只有在你要的是一個長期運作、會自己整理記憶的有狀態 agent 時**，才考慮 Letta。它是一整套 agent 執行環境，不只是記憶函式庫，導入成本比較高。

### 資料來源

- [Letta：用 Docker 部署與 PostgreSQL 設定（官方文件）](https://docs.letta.com/guides/selfhosting/postgres)
- [Letta 資料庫設定（DeepWiki，第三方原始碼整理）](https://deepwiki.com/letta-ai/letta/12.4-database-setup)
- [Claude memory tool（Anthropic 官方文件）](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
- [Context management（Anthropic 部落格）](https://claude.com/blog/context-management)
- [Vertex AI Agent Engine Memory Bank 概覽（Google Cloud 官方文件）](https://docs.cloud.google.com/agent-builder/agent-engine/memory-bank/overview)
- [Vertex AI Memory Bank 公開預覽（Google Cloud 部落格）](https://cloud.google.com/blog/products/ai-machine-learning/vertex-ai-memory-bank-in-public-preview?hl=en)
- [Agentic Memory in Production: Letta, Zep, Mem0, LangMem 比較](https://www.llms.blog/posts/agentic-memory-and-context-management-systems-in-production-comparing-letta-zep-mem0-and-langmem)
- [Best AI Agent Memory Frameworks in 2026（Atlan）](https://atlan.com/know/best-ai-agent-memory-frameworks-2026/)

> 框架比較文章多半出自相關廠商或第三方部落格，裡面的基準分數只當參考；各框架的版本細節變動很快，實際導入前以官方文件為準。
