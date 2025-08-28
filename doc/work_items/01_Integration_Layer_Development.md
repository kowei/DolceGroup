# 工作包 01：整合適配層開發 (Integration Layer)

**目標**：開發、測試並部署用於與外部 ERP 和 CRM 系統溝通的 Lambda 適配器。此階段的關鍵是建立一個穩定的抽象層，即使在外部接口規格未完全確定的情況下，也能讓內部功能先行開發。

---

### Phase 1: 接口定義與模擬 (Interface Definition & Mocking)

此階段的目標是建立一個可供內部團隊使用的、穩定的模擬接口。

- [ ] **協同接口定義**：
    - [ ] 與負責 ERP/CRM 的團隊開會，獲取初步的 API 規格文件 (spec) 或欄位對應表。
    - [ ] 根據 `design.md` 中定義的通用接口，與對方確認資料格式與欄位，並更新 `design.md` 文件。
- [ ] **建立 Mock Server**：
    - [ ] 建立一個輕量的 Mock API Server (可使用 Node.js + Express 或線上工具如 Mockoon)。
    - [ ] 在 Mock Server 上，實作 `design.md` 中定義的所有通用接口（例如 `GET /erp/order/:orderId`）。
    - [ ] 確保 Mock Server 回傳的資料格式與 `design.md` 中定義的 `Response` 一致。
    - [ ] 將此 Mock Server 部署到一個內部可存取的網路位置，供後續開發與測試使用。

### Phase 2: ERP Adapter Lambda 開發

- [ ] **IaC 資源定義**：
    - [ ] 在 `template.yaml` 中，新增 `ErpAdapterFunction` 的 `AWS::Serverless::Function` 資源定義。
    - [ ] 為此 Lambda 設定獨立的 IAM Role，並授予呼叫 Mock Server (或未來真實 ERP API) 的網路權限，以及讀取日誌的權限。
- [ ] **主控邏輯開發 (`app.js`)**：
    - [ ] 實作 Lambda 的主 handler，能解析來自內部服務的請求（例如：`{ "action": "GET_ORDER_STATUS", ... }`）。
    - [ ] 根據 `action` 欄位，路由到不同的內部處理函式。
- [ ] **API 呼叫邏輯**：
    - [ ] 開發呼叫外部 ERP API (初期為 Mock Server) 的客戶端邏輯。
    - [ ] 實作錯誤處理、重試 (retry) 與逾時 (timeout) 機制。
- [ ] **資料轉換邏輯**：
    - [ ] 建立一個轉換層 (Transformation Layer)，負責將外部 API 的回應，轉換成我們內部標準的資料格式。
- [ ] **快取機制實作**：
    - [ ] 針對不常變動的查詢（如訂單狀態），在 Lambda 中加入記憶體內快取 (in-memory cache)，並設定 TTL (Time-to-Live)，例如 5-15 分鐘。

### Phase 3: CRM Adapter Lambda 開發

- [ ] **IaC 資源定義**：在 `template.yaml` 中新增 `CrmAdapterFunction` 的資源定義與對應的 IAM Role。
- [ ] **主控與 API 呼叫邏輯**：重複 Phase 2 的步驟，實作呼叫 CRM Mock Server 的邏輯。
- [ ] **資料轉換邏輯**：建立對應的資料轉換層，確保回傳的資料符合內部標準，並過濾掉不必要的敏感個資。

### Phase 4: 測試

- [ ] **單元測試 (Unit Tests)**：
    - [ ] 針對 ERP 和 CRM Adapter 中的**資料轉換邏輯**撰寫單元測試，確保欄位對應的正確性。
    - [ ] 模擬各種來自外部 API 的回應（成功、失敗、格式錯誤），驗證轉換層的穩定性。
- [ ] **整合測試 (Integration Tests)**：
    - [ ] 撰寫測試腳本，直接觸發 `ErpAdapterFunction` 和 `CrmAdapterFunction` Lambda。
    - [ ] 驗證 Lambda 是否能成功呼叫 Mock Server，並回傳經過正確轉換的資料。
    - [ ] 測試快取機制是否如預期般運作。

### Phase 5: 部署

- [ ] **部署至開發環境**：透過 `sam deploy` 將新的 Adapter Lambdas 部署到開發環境。
- [ ] **部署後煙霧測試 (Smoke Testing)**：
    - [ ] 從 AWS Console 手動觸發 Lambda，或使用 `sam remote invoke` 指令。
    - [ ] 驗證 Lambda 在雲端環境中能成功執行，並檢查 CloudWatch Logs 中是否有非預期的錯誤。
