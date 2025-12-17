# Mermaid ダイアグラム集

LLMに「こういう図描いて」と言えばすぐ作れる。これがMermaidの強み。

## AWS構成図（実践的なパターン）

### Web3層 + RDS構成

```mermaid
flowchart TB
    subgraph Internet
        User[👤 ユーザー]
    end

    subgraph AWS Cloud
        subgraph Public Subnet
            ALB[Application Load Balancer]
            NAT[NAT Gateway]
        end

        subgraph Private Subnet - App
            ECS1[ECS Task]
            ECS2[ECS Task]
        end

        subgraph Private Subnet - DB
            RDS[(RDS Primary)]
            RDS_S[(RDS Standby)]
        end

        subgraph Shared Services
            SM[Secrets Manager]
            CW[CloudWatch]
            S3[(S3)]
        end
    end

    User --> ALB
    ALB --> ECS1 & ECS2
    ECS1 & ECS2 --> RDS
    RDS -.-> |同期レプリケーション| RDS_S
    ECS1 & ECS2 --> NAT
    ECS1 & ECS2 --> SM
    ECS1 & ECS2 --> S3
    ECS1 & ECS2 -.-> CW
```

### サーバーレス API パターン

```mermaid
flowchart LR
    Client[📱 Client] --> APIGW[API Gateway]
    
    subgraph AWS
        APIGW --> Lambda[Lambda]
        Lambda --> DynamoDB[(DynamoDB)]
        Lambda --> S3[(S3)]
        
        APIGW -.-> |認証| Cognito[Cognito]
        Lambda -.-> |ログ| CWLogs[CloudWatch Logs]
        
        DynamoDB --> |Stream| Lambda2[Lambda]
        Lambda2 --> SNS[SNS]
        SNS --> SQS[SQS]
    end
```

### マルチアカウント構成

```mermaid
flowchart TB
    subgraph Management Account
        Org[AWS Organizations]
        SSO[IAM Identity Center]
    end

    subgraph Security Account
        GuardDuty[GuardDuty]
        SecurityHub[Security Hub]
        Config[AWS Config]
    end

    subgraph Log Account
        CT[(CloudTrail Logs)]
        CWL[(CloudWatch Logs)]
    end

    subgraph Workload Accounts
        subgraph Production
            Prod[本番環境]
        end
        subgraph Staging
            Stg[検証環境]
        end
        subgraph Development
            Dev[開発環境]
        end
    end

    Org --> |管理| Production & Staging & Development
    SSO --> |認証| Production & Staging & Development
    Production & Staging & Development -.-> |ログ集約| Log Account
    Production & Staging & Development -.-> |セキュリティ監視| Security Account
```

## シーケンス図（詳細版）

### OAuth2.0 認証フロー

```mermaid
sequenceDiagram
    autonumber
    participant User as 👤 ユーザー
    participant App as 📱 アプリ
    participant Auth as 🔐 認証サーバー
    participant API as 🖥️ APIサーバー

    User->>App: ログインボタンクリック
    App->>Auth: 認証リクエスト（client_id, redirect_uri, scope）
    Auth->>User: ログイン画面表示
    User->>Auth: 認証情報入力
    Auth->>Auth: 認証処理
    Auth->>App: 認可コード発行（redirect）
    App->>Auth: トークンリクエスト（認可コード）
    Auth->>App: アクセストークン + リフレッシュトークン
    App->>API: APIリクエスト + アクセストークン
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
    PD->>L1: 通知（電話・Slack）
    
    alt 15分以内に応答あり
        L1->>PD: 応答（Acknowledge）
        L1->>L1: 調査・対応
        alt 対応可能
            L1->>PD: 解決（Resolve）
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

### Git-flow

```mermaid
gitgraph
    commit id: "init"
    branch develop
    checkout develop
    commit id: "dev-1"
    
    branch feature/login
    commit id: "feat-1"
    commit id: "feat-2"
    checkout develop
    merge feature/login id: "merge-feat"
    
    branch release/1.0
    commit id: "bump-ver"
    checkout main
    merge release/1.0 id: "v1.0" tag: "v1.0.0"
    checkout develop
    merge release/1.0

    checkout main
    branch hotfix/bug
    commit id: "fix"
    checkout main
    merge hotfix/bug id: "v1.0.1" tag: "v1.0.1"
    checkout develop
    merge hotfix/bug
```

## ガントチャート

### プロジェクトスケジュール

```mermaid
gantt
    title MkDocs導入プロジェクト
    dateFormat  YYYY-MM-DD
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
        timestamp created_at
    }

    USER {
        uuid user_id PK
        uuid team_id FK
        string email
        string name
        enum role "admin,editor,viewer"
        timestamp last_login
    }

    DOCUMENT {
        uuid doc_id PK
        uuid author_id FK
        uuid category_id FK
        string title
        text content
        enum status "draft,review,published,archived"
        timestamp created_at
        timestamp updated_at
    }

    VERSION {
        uuid version_id PK
        uuid doc_id FK
        int version_number
        text content
        uuid updated_by FK
        timestamp created_at
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
        int sort_order
    }

    COMMENT {
        uuid comment_id PK
        uuid doc_id FK
        uuid user_id FK
        text body
        timestamp created_at
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

## C4モデル（システムコンテキスト図）

```mermaid
C4Context
    title ドキュメント管理システム - システムコンテキスト

    Person(dev, "開発者", "ドキュメントを作成・編集")
    Person(reader, "閲覧者", "ドキュメントを参照")

    System(docs, "ドキュメント管理システム", "MkDocs + S3 + CloudFront")

    System_Ext(git, "Git Repository", "ソースコード管理")
    System_Ext(cicd, "CI/CD", "自動ビルド・デプロイ")
    System_Ext(idp, "IdP", "認証基盤")

    Rel(dev, git, "Push")
    Rel(git, cicd, "Trigger")
    Rel(cicd, docs, "Deploy")
    Rel(dev, docs, "Preview")
    Rel(reader, docs, "閲覧")
    Rel(docs, idp, "認証")
```

## マインドマップ

### AWSサービス分類

```mermaid
mindmap
    root((AWS))
        Compute
            EC2
            Lambda
            ECS
            EKS
        Storage
            S3
            EBS
            EFS
            FSx
        Database
            RDS
            DynamoDB
            Aurora
            ElastiCache
        Network
            VPC
            CloudFront
            Route53
            API Gateway
```

---

!!! tip "LLMとの協働"
    これらの図は全て「こういう図を描いて」と言えば生成できる。
    修正も「ここをこう変えて」と言えばすぐ対応可能。
    作図ツールでポチポチする時代は終わり！
