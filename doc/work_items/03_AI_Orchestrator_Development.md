# 工作包 03：AI 協同器開發 (AI Orchestrator)

**目標**：開發專案的「大腦」— AI Orchestrator Lambda。這是技術上最複雜的組件，負責理解使用者意圖、調度多種工具（知識庫、資料庫），並生成最終的精確回覆。

---

### Phase 1: 工具函式開發與單元測試 (Tool Implementation)

此階段的目標是建立 AI 可靠的工具箱，並確保每個工具都能獨立運作。

- [ ] **開發 `search_documents` 工具**：
    - [ ] 在 Lambda 中建立一個函式，接收 `query` 字串。
    - [ ] 實作呼叫 Amazon Kendra `Retrieve` API 的邏輯。
    - [ ] 處理 Kendra API 的成功與失敗回應。
    - [ ] 撰寫此工具的單元測試，使用 mock 來模擬 Kendra SDK 的回應。
- [ ] **開發 `query_database` 工具**：
    - [ ] 建立一個函式，接收一個結構化物件 `parameters` (例如：`{ action: 'get_price', ... }`)。
    - [ ] **[安全關鍵]** 建立一個安全的查詢產生器，根據傳入的 `action` 來選擇執行對應的、預先寫好的參數化 SQL 查詢。
    - [ ] 實作連接 AWS RDS for PostgreSQL 並執行查詢的邏輯。
    - [ ] 撰寫此工具的單元測試，驗證不同 `action` 是否能產生正確的 SQL 查詢，並模擬資料庫的回應。

### Phase 2: 協同器主流程開發 (Orchestration Logic)

- [ ] **主 Handler 開發**：
    - [ ] 建立 Lambda 的主 handler，接收來自 LINE Router Lambda 的使用者問題。
- [ ] **工具選擇提示工程 (Tool-Selection Prompting)**：
    - [ ] 設計並撰寫一個給 LLM (OpenAI) 的「系統提示」，指示它作為一個路由器，根據使用者問題和可用的工具列表，決定下一步應該呼叫哪個工具及對應的參數。
- [ ] **第一次 LLM 呼叫**：
    - [ ] 實作呼叫 OpenAI API 的邏輯，傳入上述的系統提示、工具列表和使用者問題。
    - [ ] 解析 LLM 回傳的結果，判斷出它選擇了哪個工具（`search_documents` 或 `query_database`）。
- [ ] **工具執行**：
    - [ ] 根據 LLM 的判斷，執行對應的工具函式（呼叫 Phase 1 中開發的函式）。
    - [ ] 若 LLM 判斷需使用多個工具，需實作平行呼叫 (Parallel Execution) 的邏輯，以提升效率。

### Phase 3: 回應生成 (Response Synthesis)

- [ ] **回應生成提示工程 (Response-Synthesis Prompting)**：
    - [ ] 設計並撰寫一個新的「系統提示」，指示 LLM 作為一個友善的客服，根據提供的「上下文資料」（從工具中取得的資訊）和「原始問題」，生成一段通順、完整的回答。
- [ ] **第二次 LLM 呼叫**：
    - [ ] 將從工具蒐集到的所有資訊（例如：Kendra 的文件片段、RDS 的查詢結果），連同原始問題，組合成最終的提示。
    - [ ] 呼叫 OpenAI API，生成最終的自然語言答案。
- [ ] **回傳格式化**：將 LLM 生成的純文字答案，包裝成 LINE Message Object（例如，文字訊息或 Flex Message）。

### Phase 4: 整合測試

- [ ] **單一工具路徑測試**：
    - [ ] 撰寫整合測試案例，模擬一個只會觸發 `search_documents` 的問題（例如：「退貨政策是什麼？」）。
    - [ ] 撰寫整合測試案例，模擬一個只會觸發 `query_database` 的問題（例如：「AKE001 多少錢？」）。
- [ ] **多工具路徑測試**：
    - [ ] 撰寫一個最關鍵的整合測試案例，模擬一個需要同時使用多個工具的複雜問題（例如：「我想退貨 AKE001，該怎麼做？」）。
    - [ ] 驗證整個協同器從頭到尾的處理流程是否正確無誤。

### Phase 5: 部署與雲端測試

- [ ] **IaC 資源定義**：在 `template.yaml` 中完成 `AiOrchestratorFunction` 的所有資源定義，包含 IAM 權限 (Kendra, RDS, SecretsManager) 和環境變數。
- [ ] **部署至開發環境**：透過 `sam deploy` 將 Lambda 部署到雲端。
- [ ] **端對端測試**：從 LINE 手機應用程式發送一個真實訊息，觸發整個流程，驗證是否能收到預期中的 AI 回覆。