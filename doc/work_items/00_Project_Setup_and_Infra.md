# 工作包 00：專案設定與核心基礎設施建置

**目標**：完成所有開發前的一次性設定，並透過 IaC 部署專案的核心雲端基礎設施，為後續功能開發做好準備。

---

### Phase 1: 開發環境準備 (本地端)

這些是每位後端開發者在本機都需完成的設定。

- [ ] 安裝 **AWS CLI** 並透過 `aws configure` 設定好具備管理員權限的 IAM 使用者憑證。
- [ ] 安裝 **AWS SAM CLI** (Serverless Application Model)。
- [ ] 安裝 **Docker Desktop** 並確保其在背景運行。
- [ ] 安裝 **Node.js** (v18.x 或更高版本) 及 **npm**。

### Phase 2: 雲端資源前置準備 (手動設定)

這些是有狀態、或需在專案初期手動建立一次的核心資源。

- [ ] **建立 VPC 網路環境**：
    - [ ] 登入 AWS Console，建立一個新的 VPC。
    - [ ] 在 VPC 中建立至少兩個私有子網路 (Private Subnets) 和兩個公共子網路 (Public Subnets)。
    - [ ] 建立一個 NAT Gateway，並設定路由表，確保私有子網路中的資源可以對外連線。
- [ ] **建立 RDS for PostgreSQL 資料庫**：
    - [ ] 在上述建立的 VPC 私有子網路中，建立一個 RDS for PostgreSQL 實例。
    - [ ] **[重要]** 建立一個專門給 Lambda 使用的 Security Group (例如 `lambda-db-access-sg`)，並設定其 Inbound 規則，允許來自 Lambda Security Group (下一步會建立) 的 PostgreSQL (Port 5432) 流量。
    - [ ] 記錄下資料庫的 **Endpoint Address** 和 **Security Group ID**，後續部署時會用到。
- [ ] **建立開發人員 IAM 群組**：
    - [ ] 建立一個名為 `DolceAI-Developers` 的 IAM 群組。
    - [ ] 賦予該群組開發所需的權限（例如：IAM, S3, Lambda, CloudFormation, CloudWatch 的完整存取權）。
    - [ ] 將所有後端開發人員的 IAM 使用者加入此群組。

### Phase 3: 專案初始化與首次部署 (IaC)

此階段將使用程式碼來定義與部署應用程式的無狀態基礎設施。

- [ ] **初始化 SAM 專案**：
    - [ ] 在本機執行 `sam init`，選擇 `nodejs18.x` 的 `Hello World` 範本，建立專案目錄。
    - [ ] 依照 `ops_guide.md` 的建議，建立專案的子目錄結構 (e.g., `src/handlers/ai-orchestrator/`)。
- [ ] **撰寫 `template.yaml`**：
    - [ ] 將 `ops_guide.md` v1.3 中的 `template.yaml` 完整內容複製到專案的 `template.yaml` 檔案中。
    - [ ] 建立一個給 Lambda 使用的 Security Group (例如 `lambda-sg`) 資源，並設定其 Egress 規則，允許所有對外流量 (以便存取外部 API)。
- [ ] **首次部署**：
    - [ ] 在專案根目錄執行 `sam build`。
    - [ ] 執行 `sam deploy --guided`。
    - [ ] 根據提示，輸入堆疊名稱 (`Stack Name`)、AWS 區域 (`AWS Region`)，以及先前記錄下的 RDS **Endpoint Address** 和 **Security Group ID**。
    - [ ] 等待部署完成。此步驟會自動建立 S3, Kendra, IAM Roles, API Gateway, Lambda 等資源。

### Phase 4: 部署後設定

- [ ] **設定 Secrets Manager**：
    - [ ] 登入 AWS Console，找到由 SAM 建立的 Secrets Manager Secret。
    - [ ] 手動編輯 Secret，將 `YOUR_OPENAI_API_KEY_HERE` 替換為真實的 OpenAI API Key。
- [ ] **上傳知識庫文件**：
    - [ ] 將第一批次的 FAQ、產品手冊等文件，上傳到由 SAM 建立的 S3 儲存桶 (`KnowledgeBaseBucket`) 中。
- [ ] **同步 Kendra 索引**：
    - [ ] 登入 AWS Console，找到 Kendra 索引。
    - [ ] 找到對應的 S3 Data Source，手動觸發第一次的 `Sync`，讓 Kendra 開始索引 S3 中的文件。

