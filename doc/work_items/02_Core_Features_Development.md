# 工作包 02：核心功能開發 (Core Features)

**目標**：開發除了 AI 協同器之外的其他核心業務功能 Lambda，包含訂單處理、廠商專區，以及創意發想。此階段的工作將會直接使用在「工作包 01」中建立的整合適配層。

---

### Phase 1: LINE Webhook 路由器開發

這是串接所有功能的第一個入口點。

- [ ] **IaC 資源定義**：在 `template.yaml` 中，建立 `LineWebhookRouterFunction` 的資源定義，並設定好 API Gateway 的觸發事件。
- [ ] **主控邏輯開發 (`app.js`)**：
    - [ ] 實作**簽章驗證**邏輯，確保請求來自 LINE Platform。
    - [ ] 解析 Webhook 事件的類型 (`event.type`) 與內容（例如，文字訊息、LIFF 操作的 postback action）。
    - [ ] 根據事件類型，以**非同步**方式 (`asynchronous invocation`) 呼叫下游對應的功能 Lambda（如 `AiOrchestratorFunction`, `OrderRepairFunction` 等）。
    - [ ] **[重要]** 立即回傳 `200 OK` 給 LINE Platform，避免因後端處理過久而導致 LINE 平台重試 (retry) 請求。
- [ ] **測試**：
    - [ ] 撰寫單元測試，驗證不同事件是否能正確路由到對應的目標 Lambda。
    - [ ] 使用 `sam local invoke` 進行本機整合測試。

### Phase 2: 訂單/維修功能開發 (Order/Repair Lambda)

- [ ] **IaC 資源定義**：在 `template.yaml` 中建立 `OrderRepairFunction` 的資源定義，並授予其呼叫 `ErpAdapterFunction` 和 `CrmAdapterFunction` 的 IAM 權限。
- [ ] **LIFF 接口邏輯**：
    - [ ] 建立處理 LIFF 維修申請表單提交的邏輯。
    - [ ] 驗證來自 LIFF 的請求資料格式。
    - [ ] 呼叫 `ErpAdapterFunction` 來建立維修單，並將維修碼回傳給 LIFF 前端。
- [ ] **文字查詢邏輯**：
    - [ ] 建立處理使用者文字查詢訂單/維修進度的邏輯。
    - [ ] 從事件中解析出訂單編號或維修碼。
    - [ ] 呼叫 `ErpAdapterFunction` 進行查詢。
    - [ ] 將查詢結果組合成易於閱讀的 LINE Flex Message 回傳給使用者。
- [ ] **測試**：
    - [ ] 撰寫單元測試，驗證資料驗證與 Flex Message 的產生邏輯。
    - [ ] 進行整合測試，確保能成功呼叫 Adapter Lambdas 並處理其回應。

### Phase 3: 廠商專區功能開發 (Vendor Area Lambda)

- [ ] **IaC 資源定義**：在 `template.yaml` 中建立 `VendorAreaFunction` 的資源定義，並授予呼叫 `ErpAdapterFunction` 的權限。
- [ ] **登入邏輯**：
    - [ ] 實作處理廠商輸入「廠商代碼 + 驗證碼」的邏輯。
    - [ ] 從 RDS 資料庫的 `vendor_auth` 表中，查詢並驗證代碼的正確性。
- [ ] **資料查詢邏輯**：
    - [ ] 驗證通過後，根據廠商代碼，呼叫 `ErpAdapterFunction` 查詢其專屬的訂單/維修資料。
    - [ ] 將結果組合成 Flex Message 或 LIFF 頁面所需的格式回傳。
- [ ] **測試**：
    - [ ] 撰寫單元測試，驗證登入邏輯與資料庫互動。
    - [ ] 進行整合測試，確保能正確地從 Adapter Lambda 獲取廠商專屬資料。

### Phase 4: 創意發想功能開發 (Creative AI Lambda)

- [ ] **IaC 資源定義**：在 `template.yaml` 中建立 `CreativeAiFunction` 的資源定義，並授予呼叫 DALL-E API (透過 Secrets Manager 讀取 API Key) 的權限，以及寫入 RDS `ai_generations` 表的權限。
- [ ] **API 呼叫邏輯**：
    - [ ] 實作呼叫 DALL-E API 的客戶端邏輯。
    - [ ] 處理 API 可能的錯誤與逾時。
- [ ] **資料庫寫入**：
    - [ ] 在生成圖像後，將使用者的 Prompt、回傳的圖像 URL/S3 Key、模型資訊等，存入 RDS 的 `ai_generations` 表中。
- [ ] **測試**：
    - [ ] 撰寫整合測試，驗證是否能成功呼叫 DALL-E API 並將結果存入資料庫。

### Phase 5: 部署

- [ ] **部署至開發環境**：透過 `sam deploy` 將本工作包中新增的所有 Lambda 部署到雲端。
- [ ] **部署後煙霧測試**：從 AWS Console 或 `sam remote invoke` 觸發每個函式，確保它們能正常啟動並執行基本邏輯。
