# Lecture 13：Evaluation Fundamentals & Benchmark Design 評測基礎與基準設計

- 日期：11/4（第 7 週）
- 投影片：[lecture13.pdf](../slides/lecture13.pdf)
- 必讀：[Establishing Best Practices for Building Rigorous Agentic Benchmarks](http://127.0.0.1:8329/reading/Zhu2025_Rigorous_Agentic_Benchmarks.html)

## 閱讀清單

| 類別 | 文章 | 本站譯文 | 一句話 |
|---|---|---|---|
| **必讀** | [Establishing Best Practices for Building Rigorous Agentic Benchmarks](http://127.0.0.1:8329/reading/Zhu2025_Rigorous_Agentic_Benchmarks.html) | 全譯對照 | 提出 agent 基準檢查清單 ABC，從任務有效性與結果有效性審查十個熱門 agent 基準，找出可造成最高 100% 相對誤差的缺陷，並以 CVE-Bench 示範修正後高估下降 33%。 |
| 推薦 | [How to Build Good Language Modeling Benchmarks](http://127.0.0.1:8329/reading/Press2024_How_to_Build_Good_LM_Benchmarks.html) | 全文中譯 | SWE-bench 作者 Ofir Press 的基準設計經驗：好基準要自然、能自動評分、有挑戰性，並附上單一指標、強基準線、題數等實務準則。 |
| 推薦 | [tinyBenchmarks: evaluating LLMs with fewer examples](http://127.0.0.1:8329/reading/Polo2024_tinyBenchmarks_ICML2024.html) | 全譯對照 | 借用心理測量學的試題反應理論（IRT），從 MMLU 等上萬題的基準中挑 100 題，就能把 LLM 表現的估計誤差控制在約 2%，評測成本降低兩個數量級。 |

> 必讀一定要讀；推薦、補充可以依時間和興趣挑。清單依本站 index.html 的閱讀材料整理。
