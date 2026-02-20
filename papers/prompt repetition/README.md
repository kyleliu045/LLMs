# Prompt Repetition Improves Non-Reasonong LLMs

> **💡 Meta Information**
>
> * **Venue:** `arXiv 2025` 
> * **Paper Type:** Benchmark
> * **Links:** [Paper (arXiv)](https://arxiv.org/abs/2512.14982)
> * **Institutions:** Google Research
>
> **🏷️ Domains & Tasks**
> * `Large Language Models (LLMs)` `Prompt Engineering`
> * **Core Tasks:** Prompt Optimization, Performance Evaluation

### 📌 Citation
```text
Leviathan, Y., Kalman, M., & Matias, Y. (2025). Prompt Repetition Improves Non-Reasoning LLMs. arXiv preprint arXiv:2512.14982.
```

## Abstract
> When not using reasoning, repeating the input prompt improves performance for popular models
(Gemini, GPT, Claude, and Deepseek) without increasing the number of generated tokens or latency.

### 💡 The Core Idea: Prompt Repetition (核心概念：提示詞重複)

#### 1. The Problem: Causal Attention Limitation (痛點：因果注意力限制)
* **單向注意力 (Unidirectional Attention):** LLMs 通常被訓練為因果語言模型 (Causal Language Models)，這意味著「過去的 Token 無法注意 (attend to) 未來的 Token」 。
* **順序敏感性 (Order Sensitivity):** 因為上述的架構限制，使用者輸入 Query 的「資訊排列順序」會嚴重影響模型的預測表現 。
    * *Example:* 先給文章再問問題 (`<CONTEXT> <QUESTION>`) 與先問問題再給文章 (`<QUESTION> <CONTEXT>`)，兩者的預測結果往往大不相同 。

#### 2. The Solution: `<QUERY><QUERY>` (解決方案)
* **作法:** 作者提出了一個極度簡單粗暴卻有效的解法——**Prompt Repetition (重複提示詞)** 。
* 具體來說，就是將原本的輸入 `<QUERY>`，直接複製貼上轉換為 `<QUERY><QUERY>` 。

#### 3. Why it Works (背後原理)
* 透過重複輸入，第二個 Query 中的「每一個 Token」，都能夠透過注意力機制看見第一個 Query 中的「所有 Token」 。
* 這等於巧妙地繞過了 Causal LM 的限制，讓模型在處理最後的提問時，具備了對整個 Prompt 的全局 (Global) 視野，從而解決了順序敏感性的問題 。

#### 4. The Benefits (主要優勢)
* **提升效能:** 當模型**不使用推理 (Non-reasoning)** 模式時，Prompt Repetition 能顯著提升效能 。
* **零額外成本 (Zero Overhead):** 這個技巧完全不會增加生成的輸出長度 (Lengths of generated outputs)，也不會增加延遲 (Latency) 。

### 🚀 Motivation & Practical Advantages (動機與系統工程優勢)

#### 1. The RL Intuition (從強化學習得到的直覺)
* **Observation:** 作者觀察到，經過強化學習 (RL) 訓練的推理模型，在解決問題時往往會自然而然地學會「重複使用者部分或全段的請求」。
* **Takeaway:** 既然模型本來就需要依賴這種行為來輔助理解，我們直接在 Prompt 端主動重複，就能有效幫助模型進入狀態。

#### 2. Computational Efficiency (計算效率)
這項技術在成本與速度上具有絕對的優勢，關鍵在於巧妙利用了 LLM 底層的兩階段推理架構 (Two-stage Inference)：
* **Shift to Prefill (發揮平行運算優勢):** LLM 在讀取輸入的 **預填充階段 (Prefill Stage)** 是高度可平行化的，處理速度極快。將 Prompt 在輸入端直接疊加為 `<QUERY><QUERY>`，等於把「重複資訊」的運算負擔完美轉移到這個階段，白嫖了 GPU 的平行運算紅利。
* **Zero Generation Overhead (避開解碼效能瓶頸):** 傳統作法若要求模型「先重複問題再回答」，會逼迫模型在最慢的 **解碼階段 (Decoding Stage)** 逐字生成問題，導致延遲與成本暴增。而 Prompt Repetition 直接在輸入端解決，使模型生成的 Token 數量完全不會增加，完美避開了自迴歸 (Autoregressive) 生成這個最耗費算力的效能瓶頸。

#### 3. Seamless Integration (無縫系統整合)
對於開發包含多個處理節點的閉環 (Closed-loop) Prompt 系統來說，這是一個巨大的好消息：
* **Format Invariant:** 提示詞重複不會改變模型最終生成的輸出格式，與原始 Prompt 的輸出結果完全可互換 (Interchangeable)。
* **Drop-in Deployment:** 這代表它是一個完美的「隨插即用」優化技巧。你可以在不破壞下游元件 (例如 Evaluator 的解析邏輯或 JSON Parser) 的情況下，安全且無痛地將這個技巧直接套用在任何一個節點的輸入端。

#### 4. Interaction with Reasoning (與推理模式的交互)
* 當模型已經開啟推理模式 (Reasoning enabled) 時，加入 Prompt Repetition 的效果呈現 **中性到微幅正面 (Neutral to slightly positive)**。這很合理，因為具有推理能力的大模型可能在內部思考步驟中，就已經在進行類似的重複行為了。

### 🔮 Future Directions & Variants (未來研究方向與變體探討)

作者在這部分提出了 13 個極具潛力的未來研究方向，我將其歸納為四大核心領域，並結合了 Appendix 中的實驗發現：

#### 1. 🛠️ Prompt Variants & Ablations (提示詞變體與消融實驗)
這是 Appendix 中最精彩的部分，證明了「怎麼重複」也有學問：
* **Prompt Repetition (Verbose) (囉嗦版重複):** 在兩次 Query 中間加入自然語言轉折，例如 `<QUERY> Let me repeat that: <QUERY>`。實驗顯示其效果與純粹重複差不多，有時甚至更好。
* **Prompt Repetition x3 (重複三次):** 格式為 `<QUERY> Let me repeat that: <QUERY> Let me repeat that one more time: <QUERY>`。
    * *Insight:* 在處理極度依賴大海撈針的客製化任務（如 NameIndex 或 MiddleMatch）時，**重複三次的效能會大幅超越只重複兩次 (Vanilla)**。未來值得探討何時需要 $>2$ 次的重複。
* **Padding Ablation (長度控制對照組):** 為了證明「效能提升是因為重複，而不是因為字數變多」，作者做了一個實驗：用無意義的句號 (`...`) 把原始 Prompt 墊到跟重複版一樣長。
    * *Result:* 效能完全沒有提升，證實了增益確實來自於「語意特徵的重複注意力」，而非單純的長度擴張。
* **Partial Repetition & Reordering:** 對於超長文本，未來可以嘗試「只重複 Prompt 的關鍵部分」，或者用小模型先將 Prompt 的資訊「重新排序 (Reorder)」來取代無腦重複。

#### 2. ⚡ System & KV-Cache Optimization (系統與快取底層優化)
如何讓這項技術在工程上更極致：
* **KV-Cache 截斷術:** 作者提出一個超強的工程構想：在 Prefill 階段算完兩次 Prompt 後，**「只保留第二次重複」的 KV-Cache**。這樣在後續的 Generation 階段，系統記憶體負擔完全等同於沒有重複，達成真正的「零效能折損 (Performance Neutral)」。
* **Dynamic Generation Repetition:** 在模型生成長文的過程中，「定期」把剛剛生成的 Token 再重複餵給模型，並探索其在多輪對話 (Multi-turn scenarios) 中的應用潛力。

#### 3. 🧠 Mechanistic & Architecture Synergy (底層機制與架構結合)
* **Mechanistic Interpretability (機制可解釋性):** 深入分析 Token 在「第一次出現」跟「第二次重複」時，其在模型內部的 Representation (特徵表示) 發生了什麼變化，以及 Attention 機制的注意力分佈模式。
* **與新架構結合:** 探索將 Prompt Repetition 與其他先進注意力機制（如 Selective Attention 或 Prefix LM 架構）結合使用的可能性。

#### 4. 🤖 Training & Cross-Modality (模型訓練與跨模態應用)
* **Training / Fine-Tuning:** 未來可以直接用「重複的 Prompt」來微調模型，或者訓練具備推理能力 (Reasoning) 的模型使用這個技巧，以進一步提升推理效率（模型甚至可能學會自己判斷何時該避免重複）。
* **Vision & Multi-modal:** 探討這個「重複」的技巧是否也能套用在非文本模態（如圖片 Images）上，例如輸入兩張一樣的圖片是否能提升 VLM 的辨識細節能力。
