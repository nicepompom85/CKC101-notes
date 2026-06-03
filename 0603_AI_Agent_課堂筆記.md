# AI-Agent 應用架構

> CKC101 課程筆記整理

---

## 一、AI-Agent 概述

AI-Agent（人工智慧代理）是能夠自主感知環境、做出決策並採取行動以達成目標的 AI 系統。與傳統的問答式 AI 不同，Agent 具備持續推理、規劃與工具使用的能力。

### 1.1 核心組成元素

- **LLM（大型語言模型）**：Agent 的大腦，負責理解指令、推理與生成回應
- **Memory（記憶）**：儲存對話歷史與長期資訊，使 Agent 能跨輪次保持上下文
- **Prompt（提示詞）**：驅動 LLM 行為的輸入文字，包含指令、範例與限制
- **工具（Tools）**：Agent 可呼叫的外部功能，例如搜尋、計算、API 呼叫等

### 1.2 Guardrail（護欄）

Guardrail 是對 AI-Agent 行為的安全邊界設定，防止模型產生有害、不當或偏離目標的輸出。

- 輸入過濾：偵測並攔截惡意或不當的使用者輸入
- 輸出過濾：確保模型回應符合安全規範
- 幻覺控制：結合 Grounding 技術減少虛假資訊的生成

### 1.3 審查紀錄（Audit Log）

記錄 Agent 的所有行為與決策過程，用於事後稽核、除錯與合規性驗證。包含：

- 輸入/輸出記錄
- 工具呼叫記錄
- 決策路徑追蹤（Tracing）

---

## 二、Memory（記憶機制）

記憶是讓 Agent 具備持續性與上下文感知的關鍵機制，分為不同層次：

### 2.1 記憶類型

- **短期記憶（Short-term Memory）**：當前對話的上下文視窗（Context Window）內的資訊
- **長期記憶（Long-term Memory）**：透過外部資料庫或向量儲存保留跨對話資訊
- **語義記憶**：儲存事實知識，通常以向量形式存在知識庫中

### 2.2 記憶相關概念

- **分段（Chunking）**：將長文件切割成較小的片段以利向量化處理
- **向量（Vector）**：文字的數學表示，用於語義搜尋與相似度比對
- **排序與擷取（Retrieval & Ranking）**：從向量資料庫中找出最相關的記憶片段

---

## 三、Prompt（提示詞工程）

Prompt Engineering 是設計輸入文字以引導 LLM 產生預期輸出的技術，是 AI-Agent 應用的核心能力之一。

### 3.1 Prompt 組成

- **System Prompt（系統提示）**：設定 AI 的角色、行為規範與全域指令
- **Instruction（指令）**：具體告知模型要執行的任務
- **Template（模板）**：可重複使用的結構化提示框架，包含變數佔位符

### 3.2 提示技術

- **Zero-shot**：不提供範例，直接要求模型完成任務
- **Few-shot**：提供少量範例（通常 2-5 個）來引導模型學習格式與邏輯
- **Chain of Thought（思維鏈，CoT）**：引導模型逐步思考推理過程，提升複雜任務準確度

> 💡 **注意：** Few-shot 與 CoT 結合使用效果最佳，特別適合數學推理、邏輯判斷等需要多步驟思考的任務。

### 3.3 Prompt Injection Attack（提示詞注入攻擊）

攻擊者透過惡意輸入覆蓋或竄改原始系統提示，使 AI 執行非預期行為。

- 直接注入：在使用者輸入中嵌入偽裝成系統指令的文字
- 間接注入：透過 Agent 讀取的外部資料（網頁、文件）植入惡意指令
- 防禦方式：輸入驗證、Guardrail 過濾、最小權限原則

---

## 四、Knowledge 與 Grounding（知識與基礎錨定）

### 4.1 知識來源

- **內建知識**：LLM 訓練時學習到的參數化知識（有時效截止日限制）
- **外部知識庫（RAG）**：透過檢索增強生成（Retrieval-Augmented Generation）引入即時資料
- **向量資料庫**：儲存文件的語義向量，支援高效相似度搜尋（如 Pinecone、Milvus、Weaviate）

### 4.2 Grounding（基礎錨定）

Grounding 是將 AI 回應與可驗證的事實資訊連結的技術，旨在減少幻覺（Hallucination）。

- 將回應引用到具體知識來源（文件、資料庫）
- 搭配 RAG 使用，確保回應有據可查
- 提供引用來源讓使用者可驗證

### 4.3 幻覺（Hallucination）

指 LLM 自信地生成不實或捏造資訊的問題，是目前 LLM 最主要的缺陷之一。

- 原因：訓練資料偏差、模型過度自信、上下文不足
- 緩解方法：Grounding、RAG、溫度參數調低、提示詞明確化

---

## 五、工具生態（Plugin / MCP / Skill / API）

Agent 透過各種工具介面與外部世界互動，形成完整的工具生態系統。

### 5.1 工具層次架構

- **API（應用程式介面）**：最底層的服務呼叫介面，Agent 透過 HTTP 請求與外部服務溝通
- **Plugin（插件）**：封裝 API 呼叫的可重用模組，例如 ChatGPT Plugins
- **MCP（Model Context Protocol）**：Anthropic 提出的標準化工具協議，統一 AI 與工具的溝通格式
- **Skill（技能）**：高階封裝的能力單元，結合多個工具完成特定任務

### 5.2 AoA（Agent on Agent）

AoA 是多 Agent 協作架構，一個 Agent 作為協調者（Orchestrator）呼叫其他子 Agent 執行專門任務。

- 主 Agent：負責任務規劃與子任務分配
- 子 Agent：各自擁有專屬工具與能力
- 適用場景：複雜工作流程自動化、跨系統整合

---

## 六、多 AI 協作架構

現代 AI 應用不再是單一模型，而是多個 AI 系統協同工作的生態。

### 6.1 確保輸出品質

- 評估（Evaluation）：透過評分機制衡量 Agent 輸出品質
- 成本控制：監控 Token 用量與 API 呼叫費用
- 限制長度：設定輸出長度限制避免過長回應耗費資源

### 6.2 Flow（工作流）

Flow 是將多個 AI 步驟串聯成自動化工作流程的設計模式。

- 順序流：步驟依序執行
- 並行流：多個步驟同時執行
- 條件流：根據中間結果決定執行路徑

### 6.3 AI 驅動分類

利用 AI 對輸入資料進行自動分類與路由，決定後續處理流程。

- 意圖識別：判斷使用者意圖
- 任務分派：根據分類結果分配給對應的 Agent 或工具

---

## 七、輸入處理（Input Processing）

### 7.1 輸入類型

- 文字（Text）：最常見的輸入形式
- 圖片（Image）：多模態 Agent 可處理視覺輸入
- 文件（Document）：PDF、Word、網頁等非結構化資料
- 語音（Audio）：透過 STT（語音轉文字）轉換後處理

### 7.2 輸入限制

- **限定長度（Context Limit）**：LLM 有最大 Token 數限制，過長輸入需切割或摘要
- **輸入前處理**：清洗、格式化、去除無效資訊

### 7.3 輸入前置（Input Preprocessing）

- 截斷（Truncation）：超過限制時保留最關鍵的資訊
- 摘要（Summarization）：壓縮長文為核心重點
- 正規化（Normalization）：統一格式，例如日期、數字格式

---

## 八、AI 安全與攻防

### 8.1 AI 參考（AI Reference）

AI 系統在生成回應時引用的外部資料或知識來源，用於提升回應的可信度與準確性。

### 8.2 自對抗攻擊（Adversarial Attack）

針對 AI 模型的對抗性攻擊，透過精心設計的輸入欺騙模型。

- 白盒攻擊：攻擊者掌握模型結構，可計算梯度設計攻擊
- 黑盒攻擊：僅透過輸入輸出觀察進行攻擊
- **Prompt Injection（提示注入）**：最常見的 LLM 攻擊手段（見第三章）

### 8.3 防禦策略

- 輸入過濾與驗證
- 輸出監控與審查
- 最小權限原則：Agent 僅授予必要的工具存取權
- 紅隊測試（Red Teaming）：主動尋找系統弱點

---

## 九、生成類型（Generation Types）

### 9.1 文字生成

- 創意寫作、摘要、翻譯、問答
- 結構化輸出：JSON、XML、Markdown 格式生成

### 9.2 多模態生成

- 圖片生成（Image Generation）：Stable Diffusion、DALL-E、Midjourney
- 音訊生成（Audio Generation）：TTS（文字轉語音）
- 代碼生成（Code Generation）：GitHub Copilot、Cursor

---

## 十、核心概念速查表

| 概念 | 說明 | 關鍵字 |
|---|---|---|
| Guardrail | AI 安全護欄，限制不當行為 | 安全邊界、過濾、合規 |
| Memory | 短期/長期記憶機制 | Context Window、向量、RAG |
| Prompt | 驅動 LLM 的輸入文字設計 | System Prompt、Template、CoT |
| Grounding | 將回應錨定到可驗證事實 | RAG、知識庫、幻覺防治 |
| MCP | Model Context Protocol，工具標準協議 | Plugin、API、Skill |
| AoA | Agent 呼叫 Agent 的多層協作 | Orchestrator、Sub-Agent |
| Hallucination | LLM 生成不實資訊的問題 | 幻覺、Grounding、RAG |
| Prompt Injection | 惡意輸入劫持 AI 行為的攻擊 | 安全、攻防、注入 |
| Chain of Thought | 引導逐步推理的提示技術 | CoT、Few-shot、推理 |
| Flow | 多步驟 AI 工作流自動化 | Pipeline、Orchestration |

---

*CKC101 課程筆記　|　AI-Agent 應用架構*
