# Mermaid ダイアグラム集

LLMに「こういう図描いて」と言えばすぐ作れる。これがMermaidの強み。

## AWS構成図（実践的なパターン）

### Web3層 + RDS構成

```mermaid
flowchart TB
    subgraph Internet
        User[👤 ユーザー]
    end

    subgraph AWS_Cloud[AWS Cloud]
        subgraph Public_Subnet[Public Subnet]
            ALB[Application Load Balancer]
            NAT[NAT Gateway]
        end

        subgraph Private_App[Private Subnet - App]
            ECS1[ECS Task]
            ECS2[ECS Task]
        end

        subgraph Private_DB[Private Subnet - DB]
            RDS[(RDS Primary)]
            RDS_S[(RDS Standby)]
        end

        subgraph Shared[Shared Services]
            SM[Secrets Manager]
            CW[CloudWatch]
            S3[(S3)]
        end
    end

    User --> ALB
    ALB --> ECS1
    ALB --> ECS2
    ECS1 --> RDS
    ECS2 --> RDS
    RDS -.-> RDS_S
    ECS1 --> NAT
    ECS2 --> NAT
    ECS1 --> SM
    ECS2 --> SM
    ECS1 --> S3
    ECS2 --> S3
    ECS1 -.-> CW
    ECS2 -.-> CW
```

### サーバーレス API パターン

```mermaid
flowchart LR
    Client[📱 Client] --> APIGW[API Gateway]
    
    subgraph AWS
        APIGW --> Lambda[Lambda]
        Lambda --> DynamoDB[(DynamoDB)]
        Lambda --> S3[(S3)]
        
        APIGW -.-> Cognito[Cognito]
        Lambda -.-> CWLogs[CloudWatch Logs]
        
        DynamoDB --> Lambda2[Lambda Trigger]
        Lambda2 --> SNS[SNS]
        SNS --> SQS[SQS]
    end
```

### マルチアカウント構成

```mermaid
flowchart TB
    subgraph Management[Management Account]
        Org[AWS Organizations]
        SSO[IAM Identity Center]
    end

    subgraph Security[Security Account]
        GuardDuty[GuardDuty]
        SecurityHub[Security Hub]
        Config[AWS Config]
    end

    subgraph Logging[Log Account]
        CT[(CloudTrail Logs)]
        CWL[(CloudWatch Logs)]
    end

    subgraph Workloads[Workload Accounts]
        Prod[Production]
        Stg[Staging]
        Dev[Development]
    end

    Org --> Prod
    Org --> Stg
    Org --> Dev
    SSO --> Prod
    SSO --> Stg
    SSO --> Dev
    Prod -.-> Logging
    Stg -.-> Logging
    Dev -.-> Logging
    Prod -.-> Security
    Stg -.-> Security
    Dev -.-> Security
```

## シーケンス図（詳細版）

### OAuth2.0 認証フロー

```mermaid
sequenceDiagram
    autonumber
    participant User as ユーザー
    participant App as アプリ
    participant Auth as 認証サーバー
    participant API as APIサーバー

    User->>App: ログインボタンクリック
    App->>Auth: 認証リクエスト
    Auth->>User: ログイン画面表示
    User->>Auth: 認証情報入力
    Auth->>Auth: 認証処理
    Auth->>App: 認可コード発行
    App->>Auth: トークンリクエスト
    Auth->>App: アクセストークン発行
    App->>API: APIリクエスト
    API->>API: トークン検証
    API->>App: レスポンス
    App->>User: 結果表示

    Note over App,Auth: トークン有効期限切れ時
    App->>Auth: リフレッシュトークンで更新
    Auth->>App: 新しいアクセストークン
```

### 障害発生時のエスカレーション

```mermaid
sequenceDiagram
    participant Monitor as 監視システム
    participant PD as PagerDuty
    participant L1 as L1担当者
    participant L2 as L2担当者
    participant Manager as マネージャー

    Monitor->>PD: アラート発報
    PD->>L1: 通知
    
    alt 15分以内に応答あり
        L1->>PD: 応答
        L1->>L1: 調査・対応
        alt 対応可能
            L1->>PD: 解決
        else エスカレーション必要
            L1->>PD: エスカレーション
            PD->>L2: 通知
            L2->>L1: 協力して対応
        end
    else 15分以内に応答なし
        PD->>L2: 自動エスカレーション
        PD->>Manager: CC通知
    end
```

## 状態遷移図

### デプロイパイプラインの状態

```mermaid
stateDiagram-v2
    [*] --> Pending: PRマージ

    Pending --> Building: ビルド開始
    Building --> Testing: ビルド成功
    Building --> Failed: ビルド失敗

    Testing --> DeployToStg: テスト成功
    Testing --> Failed: テスト失敗

    DeployToStg --> WaitingApproval: ステージング反映
    WaitingApproval --> DeployToProd: 承認
    WaitingApproval --> Cancelled: 却下

    DeployToProd --> Completed: 本番反映成功
    DeployToProd --> Rollback: 本番反映失敗

    Rollback --> Failed: ロールバック完了

    Failed --> [*]
    Cancelled --> [*]
    Completed --> [*]
```

### インシデント対応ステータス

```mermaid
stateDiagram-v2
    [*] --> Detected: アラート検知

    Detected --> Triaging: 調査開始
    Triaging --> Investigating: 優先度判定完了
    
    Investigating --> Mitigating: 原因特定
    Investigating --> Escalated: エスカレーション

    Escalated --> Investigating: 引き継ぎ完了

    Mitigating --> Monitoring: 暫定対応完了
    Monitoring --> Resolved: 安定確認
    Monitoring --> Mitigating: 再発

    Resolved --> PostMortem: クローズ
    PostMortem --> [*]: 振り返り完了
```

## Gitブランチ戦略

### Git-flow（フローチャート表現）

```mermaid
flowchart LR
    subgraph main_branch[main]
        M1((v1.0.0)) --> M2((v1.0.1))
    end

    subgraph develop_branch[develop]
        D1((dev)) --> D2((dev)) --> D3((dev)) --> D4((dev))
    end

    subgraph feature[feature/login]
        F1((feat)) --> F2((feat))
    end

    subgraph release[release/1.0]
        R1((release))
    end

    subgraph hotfix[hotfix/bug]
        H1((fix))
    end

    D1 --> F1
    F2 --> D2
    D3 --> R1
    R1 --> M1
    R1 --> D4
    M1 --> H1
    H1 --> M2
    H1 --> D4
```

## ガントチャート

### プロジェクトスケジュール

```mermaid
gantt
    title MkDocs導入プロジェクト
    dateFormat YYYY-MM-DD
    section 準備
        要件整理           :done, req, 2025-01-06, 3d
        技術検証           :done, poc, after req, 5d
    section 構築
        インフラ構築       :active, infra, 2025-01-14, 5d
        CI/CD構築          :cicd, after infra, 3d
        ドキュメント移行   :migrate, after cicd, 10d
    section 展開
        パイロット運用     :pilot, after migrate, 10d
        フィードバック反映 :feedback, after pilot, 5d
        全社展開           :rollout, after feedback, 5d
```

## ER図（詳細版）

### ドキュメント管理システム

```mermaid
erDiagram
    TEAM ||--o{ USER : has
    USER ||--o{ DOCUMENT : creates
    USER ||--o{ COMMENT : writes
    DOCUMENT ||--|{ VERSION : has
    DOCUMENT ||--o{ COMMENT : has
    DOCUMENT }o--o{ TAG : tagged
    CATEGORY ||--o{ DOCUMENT : contains

    TEAM {
        uuid team_id PK
        string name
        string description
    }

    USER {
        uuid user_id PK
        uuid team_id FK
        string email
        string name
    }

    DOCUMENT {
        uuid doc_id PK
        uuid author_id FK
        uuid category_id FK
        string title
        text content
    }

    VERSION {
        uuid version_id PK
        uuid doc_id FK
        int version_number
        text content
    }

    TAG {
        uuid tag_id PK
        string name
        string color
    }

    CATEGORY {
        uuid category_id PK
        uuid parent_id FK
        string name
    }

    COMMENT {
        uuid comment_id PK
        uuid doc_id FK
        uuid user_id FK
        text body
    }
```

## 円グラフ

### インシデント原因分析

```mermaid
pie showData
    title 2024年インシデント原因内訳
    "設定ミス" : 35
    "キャパシティ不足" : 25
    "外部サービス障害" : 20
    "コード不具合" : 15
    "その他" : 5
```

## システムコンテキスト図

### ドキュメント管理システム全体像

```mermaid
flowchart TB
    Dev[👤 開発者] --> |Push| Git[Git Repository]
    Reader[👤 閲覧者] --> |閲覧| Docs[ドキュメントシステム]
    
    Git --> |Trigger| CICD[CI/CD]
    CICD --> |Deploy| Docs
    Dev --> |Preview| Docs
    Docs --> |認証| IdP[IdP]
    
    subgraph DocSystem[ドキュメント管理システム]
        Docs
    end
    
    subgraph External[外部システム]
        Git
        CICD
        IdP
    end
```

## サービス分類図

### AWSサービスカテゴリ

```mermaid
flowchart TB
    AWS((AWS)) --> Compute
    AWS --> Storage
    AWS --> Database
    AWS --> Network

    subgraph Compute
        EC2
        Lambda
        ECS
        EKS
    end

    subgraph Storage
        S3
        EBS
        EFS
        FSx
    end

    subgraph Database
        RDS
        DynamoDB
        Aurora
        ElastiCache
    end

    subgraph Network
        VPC
        CloudFront
        Route53
        APIGateway[API Gateway]
    end
```

---

!!! tip "LLMとの協働"
    これらの図は全て「こういう図を描いて」と言えば生成できる。
    修正も「ここをこう変えて」と言えばすぐ対応可能。
    作図ツールでポチポチする時代は終わり！
