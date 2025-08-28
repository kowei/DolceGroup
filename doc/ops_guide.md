# 維運操作手冊 (Operations Guide) v1.3 (Node.js 完整版)

本手冊旨在提供一個清晰、可執行的指引，幫助開發者從零開始設定、建置、測試與部署本專AI整合專案的完整基礎設施（使用 Node.js）。

## 1. 環境準備 (Prerequisites)

在開始之前，請確保您的本機開發環境已安裝以下工具。所有工具都建議使用最新穩定版本。

- **AWS CLI (Command Line Interface)**
    - **用途**：用於與 AWS 服務進行互動的基礎工具。
    - **安裝指引**：[https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

- **AWS SAM CLI (Serverless Application Model)**
    - **用途**：專案的核心工具，用於建立、建置、測試與部署無伺服器應用。
    - **安裝指引**：[https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html)

- **Docker**
    - **用途**：SAM CLI 使用 Docker 容器在本機模擬 AWS Lambda 執行環境，以便進行本機測試。
    - **安裝指引**：[https://docs.docker.com/get-docker/](https://docs.docker.com/get-docker/)

- **Node.js & npm**
    - **用途**：本專案後端 Lambda 的主要開發語言與套件管理器。建議使用 Node.js 18.x 或更高版本。
    - **安裝指引**：[https://nodejs.org/](https://nodejs.org/)

## 2. 專案初始化與結構

### 2.1. 初始化專案

1.  開啟您的終端機 (Terminal) 並進入您要放置專案的工作區目錄。
2.  執行 `sam init` 指令，並依據以下建議進行選擇：
    - `Which template source?`: **1 - AWS Quick Start Templates**
    - `Choose an application template`: **1 - Hello World Example**
    - `Which runtime would you like to use?`: **nodejs18.x**
    - `Package type`: **1 - Zip**
    - `Project name`: **dolce-group-ai-service**

### 2.2. 建議的目錄結構

```
dolce-group-ai-service/
├── README.md
├── src/                                # 原始碼目錄
│   └── handlers/                       # Lambda 處理函式
│       ├── ai-orchestrator/            # AI 協同器 Lambda
│       │   ├── app.js
│       │   └── package.json
│       ├── order-handler/              # 訂單/維修 Lambda
│       │   ├── app.js
│       │   └── package.json
│       └── line-router/                # LINE Webhook 路由器
│           ├── app.js
│           └── package.json
├── template.yaml                       # SAM 核心定義檔
```

## 3. `template.yaml` 完整基礎設施配置 (Node.js)

`template.yaml` 是專案的核心，它定義了所有 AWS 資源。以下是完整的配置範例。

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2.0
Description: >
  Dolce Group - AI Integration Service (Full Infrastructure with Node.js)

Globals:
  Function:
    Timeout: 30
    Runtime: nodejs18.x
    Architectures:
      - x86_64

Parameters:
  OpenAiApiKeySecretName:
    Type: String
    Description: The name of the AWS Secrets Manager secret that holds the OpenAI API key.
    Default: dolce-group/openai-api-key

  DbSecurityGroupId:
    Type: AWS::EC2::SecurityGroup::Id
    Description: The Security Group ID for the RDS database that Lambda needs to connect to.

  DbEndpointAddress:
    Type: String
    Description: The endpoint address of the RDS database.

  DbName:
    Type: String
    Description: The name of the database in the RDS instance.
    Default: dolcegroupdb

Resources:
  KnowledgeBaseBucket:
    Type: AWS::S3::Bucket

  KendraRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: kendra.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: KendraS3AccessPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action: ['s3:GetObject']
                Resource: !Sub "arn:aws:s3:::${KnowledgeBaseBucket}/*"

  KendraIndex:
    Type: AWS::Kendra::Index
    Properties:
      Name: DolceGroupIndex
      Edition: DEVELOPER_EDITION
      RoleArn: !GetAtt KendraRole.Arn

  KendraDataSource:
    Type: AWS::Kendra::DataSource
    Properties:
      IndexId: !Ref KendraIndex
      Name: DolceGroupS3DataSource
      Type: S3
      RoleArn: !GetAtt KendraRole.Arn
      DataSourceConfiguration:
        S3Configuration:
          BucketName: !Ref KnowledgeBaseBucket

  ApiGateway:
    Type: AWS::Serverless::Api
    Properties:
      StageName: Prod

  LineRouterFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/handlers/line-router/
      Handler: app.lambdaHandler
      Events:
        LineWebhook:
          Type: Api
          Properties:
            Path: /line/webhook
            Method: post
            RestApiId: !Ref ApiGateway

  AiOrchestratorFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/handlers/ai-orchestrator/
      Handler: app.lambdaHandler
      MemorySize: 512
      VpcConfig:
        SecurityGroupIds: [!Ref DbSecurityGroupId]
      Environment:
        Variables:
          DB_ENDPOINT: !Ref DbEndpointAddress
          DB_NAME: !Ref DbName
          KENDRA_INDEX_ID: !Ref KendraIndex
          OPENAI_SECRET_NAME: !Ref OpenAiApiKeySecretName
      Policies:
        - SecretsManagerReadPolicy:
            SecretArn: !Ref OpenAiApiSecret
        - KendraQueryPolicy:
            IndexArn: !GetAtt KendraIndex.Arn

  OpenAiApiSecret:
    Type: AWS::SecretsManager::Secret
    Properties:
      Name: !Ref OpenAiApiKeySecretName
      Description: "OpenAI API Key for Dolce Group AI Service"
      SecretString: '{"api_key": "PASTE_YOUR_OPENAI_API_KEY_HERE"}'

```

## 4. 關於 RDS 資料庫等有狀態資源的管理

像 AWS RDS 這種「有狀態」的資料庫，其生命週期通常比 Lambda 等「無狀態」的運算資源更長，且包含需要永久保存的資料。因此，在 IaC 實踐中，我們通常會將其與應用程式本身分開管理。

**建議作法：**

1.  **手動或獨立腳本建立**：在專案初期，建議可透過 AWS Console **手動建立**一次 RDS for PostgreSQL 資料庫。在更成熟的環境中，則可使用獨立的 IaC 腳本（如 Terraform 或專門的 CloudFormation）來管理資料庫、VPC 等核心網路資源。
2.  **參數傳遞**：如上面的 `template.yaml` 範例所示，我們不直接在應用程式的 SAM 範本中定義 RDS 實例。而是將已建立好的資料庫的**連線資訊**（如 `DbSecurityGroupId`, `DbEndpointAddress`）作為**參數**傳遞給 SAM 範本。
3.  **安全連接**：Lambda 透過被指派到與 RDS 相同的 `SecurityGroup` 來獲得資料庫的存取權限。這是一種安全且解耦的作法。

## 5. 建置與部署流程

### 步驟 1: 安裝依賴項

在建置前，需要先為每一個 Lambda 函式安裝其 `package.json` 中定義的依賴套件。您需要進入每一個函式的目錄去執行 `npm install`。

```bash
# 範例：安裝 AI 協同器的依賴項
cd src/handlers/ai-orchestrator/
npm install
cd ../../.. # 回到專案根目錄
```

### 步驟 2: 建置專案

在專案根目錄下執行此指令。對於 Node.js，`sam build` 主要會將您的程式碼與已安裝的 `node_modules` 目錄一起複製到 `.aws-sam/build` 中，準備打包。

```bash
sam build
```

### 步驟 3: 部署專案

建置成功後，執行部署指令。首次部署建議加上 `--guided` 參數，SAM 會引導您完成設定。

```bash
sam deploy --guided
```

SAM 會詢問您一系列問題，請依序回答，例如：
- `Stack Name`: 為您的 CloudFormation 堆疊命名，例如 `dolce-group-ai-prod`。
- `AWS Region`: 您要部署的區域，例如 `ap-northeast-1`。
- `Parameter OpenAiApiKeySecretName`: [可按 Enter 使用預設值]
- `Parameter DbSecurityGroupId`: 輸入您手動建立的 RDS 資料庫所使用的 Security Group ID。
- `Parameter DbEndpointAddress`: 輸入該 RDS 資料庫的連線端點網址。
- `Confirm changes before deploy`: 建議選 `Y`，讓您在變更前能再次確認。
- `Allow SAM CLI IAM role creation`: 選 `Y`。
- `Save arguments to samconfig.toml`: 選 `Y`，方便未來重複部署。

## 6. 本機測試

在部署到雲端前，您可以使用 Docker 在本機模擬執行 Lambda，進行快速測試。

首先，建立一個模擬事件檔案，例如 `events/line_webhook_event.json`:
```json
{
  "body": "{\"destination\":\"YOUR_CHANNEL_ID\",\"events\":[{\"type\":\"message\",\"message\":{\"type\":\"text\",\"id\":\"12345\",\"text\":\"Hello, world\"},\"webhookEventId\":\"...\",\"deliveryContext\":{\"isRedelivery\":false},\"timestamp\":1625632928000,\"source\":{\"type\":\"user\",\"userId\":\"U12345...\"},\"replyToken\":\"...\",\"mode\":\"active\"}]}",
  "isBase64Encoded": false
}
```

然後，在專案根目錄執行以下指令來觸發 `LineRouterFunction`：

```bash
sam local invoke LineRouterFunction -e events/line_webhook_event.json
```

```