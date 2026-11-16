# Lecture 5：Frameworks & Orchestration 框架與編排

- 日期：10/7（第 3 週）
- 必讀：[DSPy（Khattab et al., ICLR 2024）](http://127.0.0.1:8329/reading/Khattab2024_DSPy_ICLR2024.html)
- 導讀：[Lecture 5 中文導讀](https://claude.ai/artifact/CZ49ZKe6ikibhReA5hWNUT)
- 搭配閱讀：GEPA（demo 用的優化器）、Building Effective Agents（workflow 和 agent 怎麼選）

**閱讀順序**：先讀導讀 PART 2「DSPy」（p.19–51），看程式碼怎麼寫、跑出什麼結果，再讀 DSPy 論文的 §3、§4 和一個案例研究。論文寫得比較抽象，先看 demo 再讀會順很多。LangGraph、Harness、Temporal 這幾段跟論文無關，可以之後再看。

**整講貫穿同一個例子**：從 email 抽出活動，加進行事曆 tinycal。同一件事分別用 DSPy、LangGraph、Temporal 各做一次，三個框架放在一起比最容易看出差別。

---

## 1. 框架就是抽象層（p.4–18）

### 開場為什麼用 PyTorch（p.3–5）
p.4 並排兩段程式，做的是同一件事（線性迴歸跑一步梯度下降）：

```python
# 手寫：梯度自己推、參數自己更新
d_pred = 2 * (pred - y) / y.size
d_w, d_b = np.sum(d_pred * x), np.sum(d_pred)
w, b = w - lr * d_w, b - lr * d_b

# PyTorch：框架幫你做
loss.backward()
optimizer.step()
```

這個比喻有兩個用意：
1. **說明「框架就是抽象層」**：框架會把底層細節藏起來，讓你專心在自己的問題上。這門課其他講次要你親手實作細節（例如 HW1 自己寫 harness），這一講則要你學會這些抽象。選 PyTorch 是因為大家幾乎都用過（p.3 的小調查問的就是「誰用過 PyTorch」）。
2. **替 DSPy 鋪路**：DSPy 的設計本來就是仿照 PyTorch，後面講到優化器時會再回到這個比喻（對照表見第 2 節最後的「回到開場的比喻」）。

### AI 系統的三根支柱（p.10–13）
| 資料 Data | 系統或模型 System or Model | 評測與指標 Evaluation + Metric |
|---|---|---|
| 訓練資料、rollout（執行軌跡）、示範、人類回饋… | 語言模型、複合 AI 系統／workflow、agent 系統… | 通過率、完全比對、LLM 評審、成本／延遲… |

三者都做好，才叫「AI 系統**工程**」。

### 框架地圖：依「控制權在誰手上」排列（p.14–16）
| | 模型 Model | Workflow／複合 AI 系統 | Agent |
|---|---|---|---|
| 控制流程 | 大系統裡的一小塊 | 邏輯**寫在程式裡**（我們決定） | **交給模型決定** |
| 框架 | OpenAI API、Anthropic API、**LiteLLM** | **DSPy**、**LangGraph** | OpenHands… |

橫跨整個光譜的還有：**可靠度**（Temporal）、**可觀測性**（MLflow、Langfuse、LangSmith）。

> 小結（p.18）：工程化 AI 系統需要資料、系統、評測三者；系統依自主程度從模型 → workflow → agent；光譜每一層都有框架；也有專門提升 agent 可靠度和可觀測性的框架。

> 💭 **我只認識 LiteLLM**：它在光譜最左邊的「模型」層，用同一套介面呼叫各家模型（OpenAI、Anthropic、OpenRouter…），換供應商不用改程式。它只負責「呼叫一次模型」，不管流程怎麼編排。DSPy 底層呼叫模型時就是透過 LiteLLM。

---

## 2. DSPy：簽名、模組、優化器（p.19–51）⭐

### DSPy 是論文還是方法？兩者都是
DSPy 本身是一個 **Python 框架（開源函式庫）**，必讀的那篇論文就是發表這個框架的論文。

| | 指的是什麼 |
|---|---|
| **框架／函式庫** | `pip install dspy` 裝的那個東西。demo 裡的 `dspy.Signature`、`dspy.ChainOfThought`、`dspy.GEPA` 都是它的 API |
| **論文** | Khattab et al., *DSPy: Compiling Declarative Language Model Calls into Self-Improving Pipelines*，ICLR 2024（本講必讀），介紹這個框架的設計和實驗結果 |

- 論文提出的不是一個演算法，而是一種**寫 LM 程式的方式**（programming model）：不要手寫 prompt，改成用**簽名**宣告輸入輸出、用**模組**組合流程，再讓**優化器**（論文裡叫 teleprompter）根據資料和指標，自動「編譯」出好的 prompt 或權重。
- 名字常被解讀成 **D**eclarative **S**elf-improving **Py**thon：宣告式、能自我改進、用 Python 寫。
- **GEPA、MIPROv2** 是 DSPy 裡的**優化器（方法）**，各自有自己的論文，可以把它們想成 PyTorch 裡的 Adam、SGD。
- **LangGraph** 和 DSPy 同類，都是框架，但沒有對應的學術論文。
- 讀論文時：§3 對應「簽名 + 模組」，§4 對應「優化器」，可以直接對照下面的行事曆 demo。

### 三個抽象（p.20–22）
| 抽象 | 回答什麼 | 做什麼 |
|---|---|---|
| **簽名 Signature** | 做**什麼**（What） | 嚴格定義語言模型的輸入和輸出 |
| **模組 Module** | **怎麼**做（How） | 把簽名和程式邏輯組合成可呼叫的函式 |
| **優化器 Optimizer** | 怎麼變好 | 改進程式的 prompt 或權重，讓指標最大化 |

### 簽名：只寫型別，prompt 由框架產生（p.23–28）
```python
dspy.Predict("message -> response")                       # 最簡單的字串簽名
dspy.Predict("message: str -> sentiment: Literal['positive','negative','neutral'], confidence: float")

class ExtractEvent(dspy.Signature):                        # 複雜時改用類別
    """Extract event details from an email."""
    email: str = dspy.InputField()
    event_name: str | None = dspy.OutputField()
    date: dt.date | None = dspy.OutputField()
    start: dt.time | None = dspy.OutputField(desc="24-hour clock")
    duration_minutes: int | None = dspy.OutputField()
```
- 沒加型別時，輸出大小寫不一致（`positive` / `Negative`）；加上 `Literal` 後輸出只會是列出的三個值之一。
- 和 Lecture 2 的結構化輸出、Lecture 4 的 JSON schema 是同一件事：**用型別約束模型的輸出**。

### 模組：換推論策略、組合多次呼叫（p.29–34）
- `dspy.Predict` 換成 `dspy.ChainOfThought`，**簽名不用改**，輸出會多一個 `reasoning` 欄位。
- 用 `dspy.Module` 把兩次 LLM 呼叫和普通 Python 邏輯接起來：

```python
class UpdateCalendarFromEmail(dspy.Module):
    def __init__(self):
        self.extract = dspy.ChainOfThought(ExtractEvent)   # LLM 呼叫 1
        self.pick_emoji = dspy.ChainOfThought(PickEmoji)   # LLM 呼叫 2

    def forward(self, email):
        details = self.extract(email=email)
        if details.event_name is None or details.date is None or details.start is None:
            return dspy.Prediction(event=None, status="no event found in email")  # ← 我們寫的 if
        emoji = self.pick_emoji(event_name=details.event_name).emoji
        event = tinycal.add_event(...)
        return dspy.Prediction(event=event, status="added to calendar")
```
這是 **workflow**：要不要加進行事曆，由**我們寫的 if** 決定，不是模型決定。

### 優化器：GEPA 讓小模型從 55% 變 100%（p.35–44）
| 步驟 | 內容 |
|---|---|
| ① 資料 | 50 封擬真 email，每封附標準答案；切成 **30 筆訓練、20 筆測試** |
| ② 指標 | `event_metric`：**日期、開始時間、長度三者全對**才得 1 分，否則 0 分（活動名稱是自由文字，不評分）。同時回傳**文字回饋**，例如「date: expected …, got …」 |
| 驗證 | 大模型在測試集上 20/20 = 100%，代表任務本身做得到 |
| ③④ 小模型基準 | 同一個簽名換成 Gemma 3 12B，優化前 11/20 = **55%** |
| ⑤ 優化 | `dspy.GEPA(metric=event_metric, max_metric_calls=100, reflection_lm=大模型)`，約 3 分鐘 |
| ⑥ 結果 | 小模型 20/20 = **100%** |

- **大模型負責反思小模型錯在哪，被改的是小模型用的 prompt。**
- **GEPA 只改了簽名的指令文字**（`signature.instructions`），沒動權重，也沒動程式邏輯。改完的指令很長，寫了日期格式要用 YYYY-MM-DD、沒寫時區就用 UTC、常見陷阱等。
- 這就是 prompt 版的 `optimizer.step()`：不用手改 prompt，只要給資料和指標，框架會自己去改。

### DSPy 裡的 agent（p.45–50）
- `dspy.ReAct("request -> summary", tools=tinycal.TOOLS)`：給簽名和工具清單，一行就能做出 agent。**要呼叫哪些工具、呼叫幾次，由模型自己決定。**
- `dspy.RLM`（Recursive Language Models）：長文不直接塞進模型，而是放進 Python 環境當成一個變數，讓模型寫程式去切、去讀，必要時再遞迴呼叫子模型處理片段。可以對照 Lecture 4 的 Lost in the Middle。

### DSPy 小結（p.51）
簽名定義輸入輸出契約 → 模組決定怎麼完成（`Predict`、`ChainOfThought`、`BestOfN`…）→ 有資料和指標時，優化器改進 prompt 或權重 → agent 用 `ReAct`、`RLM`。

### 回到開場的比喻：DSPy 就是「prompt 版的 PyTorch」
看完 demo 再回頭對照，DSPy 和 PyTorch 的概念可以一對一對上（DSPy 論文也明確說設計受 PyTorch 啟發）：

| PyTorch（訓練神經網路） | DSPy（寫 LM 程式） |
|---|---|
| `nn.Module` + `forward()` | `dspy.Module` + `forward()`（demo 的 `UpdateCalendarFromEmail`） |
| 層（`nn.Linear`、`nn.Conv2d`） | 模組（`dspy.Predict`、`dspy.ChainOfThought`） |
| 被調整的是**權重** | 被調整的是 **prompt**（指令、示範），也可以是權重 |
| loss function | 指標（demo 的 `event_metric`） |
| 訓練資料 | 訓練集（30 封 email） |
| `optimizer.step()`（SGD、Adam） | 優化器（GEPA、MIPROv2） |

**為什麼這個比喻重要**：沒有 DSPy 的時候，改進 LLM 系統只能手動調 prompt，試一下、看結果、再改，就像沒有自動微分的年代要手算梯度。DSPy 想做的，是讓「改 prompt」也變成**可以自動優化的步驟**。你不會手動調權重，交給 optimizer；同樣地，你也不必手改 prompt，給資料和指標，交給 GEPA。

---

## 3. LangGraph：狀態、節點、邊（p.52–77）

| 元素 | 是什麼 |
|---|---|
| **狀態 State** | 整張圖共用的工作區，執行過程中不斷被修改 |
| **節點 Node** | 函式，讀取狀態，**只回傳自己改動的欄位**，再由 LangGraph 合併回狀態 |
| **邊 Edge** | 控制流程；**條件邊**依狀態決定下一步往哪走 |
| Start / End | 每張圖都要有起點和終點 |

### 條件邊：沒有活動就直接結束（p.66–72）
```python
def route(state) -> Literal["pick_emoji", "__end__"]:
    d = state["details"]
    return "pick_emoji" if d.event_name and d.date and d.start else END

builder.add_edge(START, "extract")
builder.add_conditional_edges("extract", route)   # ← 判斷寫在這裡
builder.add_edge("pick_emoji", "add_to_calendar")
builder.add_edge("add_to_calendar", END)
```
- agent 版：`create_agent(llm, tools=tinycal.TOOLS)`，圖裡只有 LLM 和 Tools 兩個節點，兩者之間循環執行。

### DSPy vs. LangGraph（導讀依 demo 整理）
| | DSPy | LangGraph |
|---|---|---|
| 定義輸入輸出 | Signature | Pydantic + `with_structured_output` |
| 組合流程 | Module 的 `forward`（寫 Python） | 節點 + 邊（畫圖） |
| 分支 | `if` | 條件邊 |
| 做 agent | `dspy.ReAct` | `create_agent` |
| 自動改進 | 優化器（GEPA 等） | 投影片沒提 |

> 回顧（p.77）：**控制流程由我們設計，並決定在哪些地方讓模型做決定。** 要改進系統就需要資料和評測，DSPy 用優化器把這件事寫進框架。

> 💭 **LangGraph 是用來建立複合 AI 系統的嗎？** 對，它位在光譜中間的 workflow／複合 AI 系統。它用「圖」把多次 LLM 呼叫、工具和程式邏輯串起來，也能用 `create_agent` 做出 agent。
>
> 💭 **Google ADK（Agent Development Kit）也是這類嗎？**（投影片沒提，補充）是。ADK 是 Google 開源的 agent 框架，跟 LangGraph、CrewAI 同類，可以定義固定流程（依序、平行、迴圈），也可以把決定權交給 LLM。各家框架提供的抽象不同，底層概念都一樣：**哪些步驟由程式決定、哪些交給模型決定**。

### 延伸討論：workflow 裡誰負責什麼？什麼時候才需要 tool？

這段從 p.77「控制流程由我們設計」延伸出來，用一個實際情境整理：**網站上要用 LLM 翻譯文字，再組合、做後續邏輯，這樣需要寫 tool 嗎？**

**先看 demo 裡的分工**（DSPy 版的 `forward`）：

```python
def forward(self, email):
    details = self.extract(email=email)        # LLM：從信裡抽出活動（理解文字）
    if details.date is None: return ...        # 你的程式：判斷要不要繼續
    emoji = self.pick_emoji(...)               # LLM：挑 emoji（需要語意判斷）
    tinycal.add_event(...)                     # 你的程式：直接呼叫函式寫進行事曆
```

| 誰做 | 做什麼 |
|---|---|
| **LLM** | 只做程式寫不出來、需要「理解語言」的步驟：抽取、分類、翻譯、摘要、生成 |
| **你的程式** | 其他全部：決定流程、判斷分支、呼叫 API、寫資料庫、組合結果 |

- 在 workflow 裡，`tinycal.add_event(...)` 是**你直接呼叫的普通函式，不是 tool**。
- **只有讓模型決定要不要呼叫、要呼叫哪個時，才需要 tool**，也就是 agent（`dspy.ReAct(..., tools=...)`、`create_agent(...)`）。workflow 的流程是你寫死的，要呼叫什麼就直接呼叫。

**回到翻譯情境：不需要 tool，是標準的 workflow**

```
使用者文字 ──► [LLM 翻譯] ──► [你的程式：組合] ──► [你的程式：後續邏輯]
               唯一的 LLM 步驟      普通程式碼            普通程式碼
```

LLM 那一步要做的是**限制輸出**，讓後面的程式可以穩定接手：

1. **System prompt**：說明任務和規則（目標語言、語氣、專有名詞不翻、只輸出譯文）。
2. **結構化輸出**（最重要）：要求回 JSON 或用 schema 約束，程式拿到的才是乾淨的欄位。跟 demo 裡用 `Literal[...]` 解決大小寫不一致是同一個道理。
3. **術語表**：有固定譯名就直接放進 prompt。這是「給資料」，不需要 tool。

```python
# 一般寫法（任何 SDK 都可以）
SYSTEM = """你是翻譯器。把使用者的文字翻成繁體中文。
規則：產品名稱保留英文；不要加任何解釋。
只輸出 JSON：{"translation": "..."}
固定譯名：checkout=結帳、cart=購物車"""

def translate(text: str) -> str:
    reply = llm([{"role": "system", "content": SYSTEM},
                 {"role": "user", "content": text}])
    return json.loads(reply)["translation"]     # 失敗要處理：重試或退回原文

def build_page(items):
    translated = [translate(t) for t in items]  # LLM 步驟
    return combine(translated)                  # 你的程式：組合與後續邏輯
```

```python
# DSPy 寫法：簽名就等於 schema 加 prompt，框架會幫你產生 prompt
class Translate(dspy.Signature):
    """Translate to Traditional Chinese. Keep product names in English."""
    text: str = dspy.InputField()
    translation: str = dspy.OutputField()

translate = dspy.Predict(Translate)
```

**實作時套用第 6 節的最佳實踐**
- **保持簡單**：這個情境用 workflow 就夠，不需要做 agent。
- **讓失敗看得見**：LLM 呼叫要設逾時和重試；JSON 解析失敗時要有備案（重試一次，再失敗就顯示原文）。
- **快取**：同一段文字常會重複出現，翻過的結果存起來，省錢也省時間。
- **評測**：準備幾十筆「原文 → 理想譯文」，改 prompt 或換模型前後各跑一次比較。有了這份資料，之後也能用 DSPy 優化器自動調 prompt。

**什麼時候才需要 tool？** 要由**模型自己判斷**要不要去查東西時才需要。例如「看到不認識的專有名詞就查公司的術語資料庫」，而且術語多到放不進 prompt。即使是這樣，也常可以改成由你的程式先檢索、再放進 prompt（Lecture 3 的 RAG），流程照樣維持 workflow。

> **一句話判斷**：流程固定 → workflow，LLM 只是其中一步，用 prompt + 結構化輸出限制它；流程要由模型臨場決定 → agent，才需要把函式包成 tool。

---

## 4. 開源 Agent Harness：同一個模型，外殼不同，結果就不同（p.78–89）

**Harness** 是包在 LLM 外面的程式，負責把規劃、記憶、工具、環境接起來。設計時要決定四件事：**工具、記憶、上下文管理、執行方式**，並在成功率、成本、延遲、使用者控制權之間取捨。

| Harness | 設計特色 |
|---|---|
| Mini SWE-Agent | 只有 bash；歷史完全線性；每個動作獨立執行，不保留 shell 狀態 |
| OpenHands | 每一步存成事件；持續存在的終端機；同樣的動作重複 4 次就自動停下 |
| Pi Agent | 只有 read/write/edit/bash 四個工具；歷史存成樹，可以回到某一步再分支；沒有權限系統 |
| OpenClaw | 身分和記憶存在 Markdown 檔；每晚整理記憶；每 30 分鐘自動醒來一次（heartbeat） |

- **Harness 影響很大**（p.86，Terminal-Bench 2）：同一個模型換 harness，分數會變。**模型越小影響越大**，Haiku 4.5 在不同 harness 間差了 16.7 個百分點。
- **Meta-Harness**（p.87–88）：讓 agent 讀舊候選的程式碼、執行軌跡和分數，提出新的 harness，再評分，反覆循環。找到的 harness 勝過所有人寫的版本。
- 結論（p.89）：**做 agent 沒有唯一正解。**

> 💭 **Harness 要自己寫，還是讓 AI 寫？** 兩種都有，前提不同：
> - **自己寫**：HW1「Build an Agentic Harness」就是要你自己寫。先自己寫一個簡單的，才知道每個設計選擇會影響什麼。
> - **讓 AI 寫**：Meta-Harness 證明 AI 可以搜出更好的 harness，但它**需要一套評測和分數**才能判斷哪個比較好。思路和 DSPy 優化器一樣，只是被優化的東西從 prompt 換成整份 harness 程式碼。
> - 沒有評測時，AI 改得好不好你無從判斷，所以要先有評測，再談讓 AI 優化。

### HW1《Build an Agentic Harness》是什麼

- 發布 10/5（Lecture 4 當天），**10/30 截止**，份量很長。講義和 starter 在 [hw/cs329z_assignment1_harness.pdf](../hw/cs329z_assignment1_harness.pdf)、[hw/assignment1-harness.zip](../hw/assignment1-harness.zip)。
- 情境：替虛構公司 **Cardinal Energy** 的員工做一個工作助理 agent，能查 email（約 58 MB 的信件庫）和公司文件（員工手冊、報帳政策、產品規格等）。
- **重點限制**：模型只提供「訊息進、文字出」，**沒有內建 function calling**。工具呼叫的格式、解析、執行全都要自己寫，也就是自己做一個 harness。

| 部分 | 題目 | 要做什麼 | 對應講次 |
|---|---|---|---|
| **Part 1 LLM pipelines**（workflow） | priority | 把 email 分成 urgent／normal／ignore（正確率 ≥ 85%） | L2 prompt |
| | daily_digest | 把一天的信整理成 300 字以內的晨報 | L2 |
| | bm25 | 自己實作 BM25 檢索（公式要精確到 1e-6，速度也有要求） | L3 RAG |
| | email_qa | 最多搜 3 次（多跳），回答問題並附上證據信件的 id | L3 RAG |
| **Part 2 Cardinal Agent**（agent） | tool_registry | 工具註冊表：驗證參數（JSON Schema），**執行永遠不丟例外** | L4 工具 |
| | tool_protocol | 自己設計工具呼叫的文字格式：寫進 prompt、解析模型回覆（被截斷、多個呼叫等都要處理） | L4 function calling |
| | agent_loop | 主迴圈：回合上限、token 預算、連續 3 次格式錯誤就停、留下事件紀錄 | **L5 harness** |
| | terminal | 讓 agent 在沙箱裡跑 shell 指令 | L5 執行方式 |
| | compaction | 上下文超過預算時用摘要壓縮 | L5 上下文管理 |
| | memory | 跨對話的持久記憶（remember／recall，存成檔案） | L5 記憶 |
| | interactive | confirm／auto 模式：會改東西的工具執行前要先問使用者 | L5 使用者控制權 |
| | guardrails | 防護（例如提示注入） | L4 護欄 |
| | sim_eval | 用模擬使用者（persona）和 LLM 評審評測整個 agent | L5 評測 |

- 對照 Lecture 5 講的 harness 四個設計選擇：**工具**（registry、protocol）、**記憶**（memory）、**上下文管理**（compaction）、**執行方式**（agent_loop、terminal、interactive），HW1 每一項都要自己做。
- 評分方式：**確定性測試**用 `ScriptedLM`（照劇本回覆的假模型，不花錢）測錯誤處理；**live 測試**呼叫真模型，分數要過門檻。另外還有一份設計備忘錄（design memo）。
- 本機執行：`uv run python data/download.py` 解壓信件，再跑 `uv run pytest`（starter 每個測試都會先失敗，回傳 `NotImplementedError`）。

### 最小 harness 範例（概念示意，不是 HW1 解答）

一個 harness 的核心只有三塊：**① 工具註冊、② 文字協議（教模型怎麼寫呼叫、再解析回來）、③ 迴圈**。下面這段約 40 行，可以直接用 `python3` 跑，用假模型示範一輪「呼叫工具 → 拿到結果 → 回答」：

```python
import json, re

# ① 工具：名稱、說明、要執行的函式
TOOLS = {
    "add": {"desc": "把兩個數字相加，參數 a、b", "fn": lambda a, b: a + b},
    "today": {"desc": "回傳今天日期，不用參數", "fn": lambda: "2026-10-07"},
}

def execute(name, args):
    """執行工具。不管發生什麼都不丟例外，把錯誤當成結果回給模型。"""
    if name not in TOOLS:
        return f"錯誤：沒有 {name} 這個工具"
    try:
        return str(TOOLS[name]["fn"](**args))
    except Exception as e:
        return f"錯誤：{e}"

# ② 協議：在 prompt 裡告訴模型怎麼「用文字」呼叫工具
SYSTEM = "你可以使用以下工具：\n" + "\n".join(
    f"- {n}：{t['desc']}" for n, t in TOOLS.items()
) + '\n要呼叫工具時，只輸出一行：<tool>{"name": "...", "args": {...}}</tool>\n不需要工具時，直接回答。'

def parse(reply):
    """從模型回覆中找出工具呼叫。沒有呼叫回傳 None；格式壞了回傳錯誤訊息。"""
    m = re.search(r"<tool>(.*?)</tool>", reply, re.S)
    if not m:
        return None, None
    try:
        call = json.loads(m.group(1))
        return (call["name"], call.get("args", {})), None
    except Exception as e:
        return None, f"工具呼叫格式錯誤：{e}"

# ③ 迴圈：呼叫模型 → 解析 → 執行工具 → 把結果放回對話 → 再呼叫模型
def run_agent(lm, user_msg, max_turns=5):
    messages = [{"role": "system", "content": SYSTEM},
                {"role": "user", "content": user_msg}]
    for _ in range(max_turns):                         # 執行預算：最多呼叫幾次模型
        reply = lm(messages)
        messages.append({"role": "assistant", "content": reply})
        call, error = parse(reply)
        if error:                                      # 格式壞了：把錯誤丟回去讓模型重來
            messages.append({"role": "user", "content": error})
        elif call is None:                             # 沒呼叫工具 = 最終答案
            return reply
        else:
            result = execute(*call)
            messages.append({"role": "user", "content": f"工具結果：{result}"})
    return "超過最多回合數，停止"

# 用假模型跑一次（照劇本回覆，不花錢；HW1 測試裡的 ScriptedLM 也是這個概念）
script = iter(['<tool>{"name": "add", "args": {"a": 2, "b": 3}}</tool>', "2 + 3 = 5"])
print(run_agent(lambda messages: next(script), "2 加 3 是多少？"))   # → 2 + 3 = 5
```

**跑起來的對話長這樣**：
1. system：工具清單 + 呼叫格式
2. user：「2 加 3 是多少？」
3. assistant：`<tool>{"name": "add", "args": {"a": 2, "b": 3}}</tool>`
4. user：「工具結果：5」（harness 執行工具後塞回去的）
5. assistant：「2 + 3 = 5」→ 沒有工具呼叫，迴圈結束

**這個範例故意沒做、但 HW1 要做的**（每一項都是一個設計選擇）：
- **參數驗證**：用 JSON Schema 先檢查參數，不合格就不執行（tool_registry）。
- **格式更嚴謹**：回覆被截斷、一次出現兩個呼叫、參數字串裡又剛好包含 `<tool>` 時怎麼辦（tool_protocol）。上面用 `<tool>` 標籤只是一種做法，格式由你設計。
- **停止條件更完整**：token 預算、連續 3 次格式錯誤就停（agent_loop）。
- **事件紀錄**：每一步存成不能修改的 transcript，類似 OpenHands 的事件流。
- **上下文爆掉時壓縮、跨對話記憶、執行前請使用者確認、防提示注入**。

> 對照 DSPy：`dspy.ReAct(簽名, tools=...)` 一行就做出 agent，因為框架幫你把上面這三塊寫好了。HW1 就是把這一行背後的東西自己寫一遍。

---

## 5. 編排與遙測：讓長時間任務不怕當機、出事查得到（p.90–122）

**為什麼需要**：agent 能做的任務越來越長（METR 的圖：已經到人類要花好幾小時的任務）。跑了三小時、在最後十分鐘當機，沒有編排就得全部重來，前面 LLM 呼叫的錢也白花了。**有日誌和觀測工具才好除錯。**

### Temporal 的執行模型（p.94–98）
| 元件 | 誰寫的 | 做什麼 |
|---|---|---|
| **Activity** | 我們 | 可能失敗的單一步驟，例如一次 LLM 呼叫、一次 API 呼叫 |
| **Workflow** | 我們 | 把多個 activity 串起來 |
| **Worker** | 我們 | 真正執行 workflow 和 activity 的程式 |
| **Server** | Temporal | **保存執行歷史和狀態**，用任務佇列把工作派給 worker |

```python
RETRY = RetryPolicy(initial_interval=timedelta(seconds=1), maximum_attempts=5)

@workflow.defn
class CalendarWorkflow:
    @workflow.run
    async def run(self, email):
        details = await workflow.execute_activity(extract_event, email,
                      start_to_close_timeout=timedelta(minutes=1), retry_policy=RETRY)
        if details is None:
            return "no event found in email"
        emoji = await workflow.execute_activity(select_emoji, ...)
        return await workflow.execute_activity(add_to_calendar, ...)
```
- 邏輯和 DSPy 版幾乎一樣，差在每次呼叫外面多包了「**逾時 + 重試**」，而且結果會被記錄。
- **workflow 本身必須是確定性的**：不能自己呼叫 LLM 或做 I/O，因為 Temporal 會照記錄重播它。結果會變動的事都要放進 activity。
- **重試**（p.107–111）：故意讓 activity 前兩次回 503，server 依 RetryPolicy 自動重試，第三次成功，程式不用自己寫 try/except 迴圈。
- **殺掉 worker 也不會丟**（p.112–117）：workflow 睡 20 秒等提醒時間，第 10 秒時把 worker 殺掉，狀態仍是 RUNNING。重啟 worker 後，**LLM 步驟沒有重跑**，行事曆上也**只有一筆**活動。

> 💭 **Temporal 是 LLM 嗎？還是平台？**
> **都不是 LLM。Temporal 是開源的「可持久執行（durable execution）」編排平台**，也就是 workflow 引擎，本來是給一般分散式系統用的（例如付款、訂單流程），跟 AI 無關。它不會思考，只負責讓程式「當機也不會從頭來」。這講把它套在 agent 上，是因為 agent 也是要跑很久、每一步都可能失敗的程式。
>
> 💭 **LLM 步驟不重跑，是因為用了 Temporal 嗎？** 對。原因有兩個：
> 1. 每個 activity 完成後，**結果存在 Temporal server，不在 worker 的記憶體裡**。worker 死掉不會丟資料。
> 2. 新 worker 起來時會**照記錄重播** workflow：已經有結果的 activity 直接拿記錄的值，不會真的再執行一次，所以不會再呼叫 LLM、也不會再加一筆活動。連計時器（`workflow.sleep`）也由 server 計時。
>
> 這也是為什麼 workflow 必須是確定性的：重播時要能走出和第一次一模一樣的路徑。

### 觀測工具（p.118–121）
| 工具 | 用途 |
|---|---|
| MLflow | 追蹤每次 DSPy 呼叫，看實際送出的 prompt 和輸出 |
| LangSmith | LangChain／LangGraph 官方的觀測平台 |
| Langfuse | 儀表板：延遲、成本、trace 數、評分 |

> 小結（p.122）：**編排讓 agent 可靠，可觀測性讓你能除錯和監控。** 評測、監控和可持久執行都做好了，才算「工程化」agent，只做到能跑還不算。

---

## 6. 最佳實踐與挑戰（p.123–132）

1. **保持簡單**（p.124）：不一定要做 agent，有時 workflow 才是對的。能用光譜左邊解決，就不要往右走。
2. **讓失敗看得見、救得回來**（p.125–126）
   - 追蹤模型呼叫、工具呼叫、狀態變化和錯誤。
   - 設定逾時、重試上限和執行預算。
   - 記錄輸出、診斷失敗、監控系統狀態。
   - 可以把錯誤丟回給 agent 讓它再試。例子：Shepherd 用一個 meta-agent 監看另一個 agent，可以攔截錯誤或回滾。
3. **跑評測**（p.127–128）：**做設計決策最好的方式，是以評測為依據。評測也讓優化成為可能。** 要建自己領域的評測，不要只刷通用 coding 基準。
4. **挑戰**（p.129）：抽象會讓你離底層更遠，所以**要把 prompt 印出來看**（例如 GEPA 優化完要看 `signature.instructions`）；系統一複雜，就一定要看 trace；用了框架就多了外部依賴。

> 💭 **評測的對象是 workflow 還是 agent？要怎麼評？讓 AI agent 評嗎？**
> - **兩者都要評**，評的是「整個系統在你的任務上表現好不好」。系統做成 workflow 或 agent 都一樣，同一份評測還能拿來比較該用哪一種，這就是「用評測做設計決策」。
> - **怎麼評**＝三根支柱裡的「資料 + 指標」：準備一批有標準答案的任務（demo 的 50 封 email），再定義指標。
> - **指標不一定要用 AI**：
>   - 答案能比對時，用**程式判斷**最可靠，例如 demo 的 `event_metric`（三個欄位全對才得分）、Lecture 4 的 AST 比對。
>   - 答案是自由文字、很難比對時，才用 **LLM 評審**（三根支柱表裡列的那一項），但評審本身也要驗證過才能信。
> - 除了準確率，也要看成本、延遲，還要涵蓋邊界情況和失敗情境。延伸閱讀：Demystifying Evals。

**總結**（p.130–132）：工程化 agent 是一個循環：**建構 → 執行 → 評測 → 改進**。從簡單開始，加複雜度要有理由；為失敗和復原做設計；用證據判斷品質、成本和延遲；要完全理解自己的系統怎麼運作。

---

## 自我檢查（答得出來就可以進 Lecture 6）

**Q1. DSPy 的簽名、模組、優化器各負責什麼？GEPA 在 demo 裡改了程式的哪個部分？**（p.22、p.42–44）
- 簽名：定義模型的**輸入和輸出**（做什麼）。
- 模組：把簽名和程式邏輯**組合成可呼叫的函式**，並決定推論策略，例如 Predict、ChainOfThought（怎麼做）。
- 優化器：給定**資料和指標**，改進 prompt 或權重，讓指標最大化。
  - 這裡的「指標」是**我們自己定義的評分函式**。demo 的 `event_metric` 規定日期、開始時間、長度三者全對得 1 分，否則 0 分，另外附上文字回饋給 GEPA 參考。
- GEPA **只改了簽名的指令文字（instructions）**，沒動模型權重，也沒動 `forward` 的程式邏輯。

**Q2. 同一個「沒有活動就不加」的判斷，DSPy 和 LangGraph 各寫在哪裡？**（p.32、p.69–70）
- DSPy：寫在 Module 的 **`forward` 裡的 `if`**。
- LangGraph：寫成一條**條件邊**（`route` 函式 + `add_conditional_edges`）。
- （Temporal 版也一樣，寫在 workflow 的 `if details is None`。）
- 這三種寫法都是**我們寫的程式在做判斷**，不是模型，所以它們都是 workflow。

**Q3. Temporal 的 worker 被殺掉再啟動，為什麼 LLM 步驟不會重跑、活動也不會重複加入？**（p.102、p.114–115）
- 每個 activity 的結果都**記錄在 Temporal server 上**，不在 worker 裡。
- 新 worker 起來後**照記錄重播** workflow，已完成的 activity 直接拿記錄的結果，不會真的再執行，所以不會再呼叫 LLM，也不會再加一次活動。
- 計時器也由 server 管理，worker 死掉期間 workflow 仍是 RUNNING。

---

## 待辦
- [ ] 讀 DSPy 論文 §3（程式模型：簽名、模組）、§4（優化器／teleprompter），再挑一個案例研究，對照本講的 demo
- [ ] 選讀：GEPA、Building Effective Agents、Demystifying Evals
- [ ] 跑一次 [lecture05-code.zip](../slides/lecture05-code.zip) 的 notebook（約 1 美元以內的 OpenRouter 額度）
