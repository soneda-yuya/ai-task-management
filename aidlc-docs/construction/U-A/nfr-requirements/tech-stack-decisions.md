# U-A Infrastructure — 技術スタック決定 (Tech Stack Decisions)

**ユニット**: U-A Infrastructure
**作成日**: 2026-04-19

---

## 1. クラウドプロバイダとコアサービス

### 1.1 GCP(バックエンド側)

| サービス | 用途 | プロジェクト全体方針との整合 |
|---|---|---|
| **Cloud Run** | Core API(Go), AI Agent Service(Python)のサーバレスコンテナホスティング | execution-plan.md、application-design.md の選定通り |
| **Cloud Run Jobs** | AI Agent のサンドボックス実行(PR 生成時の隔離コンテナ) | NFR-S-07 サンドボックス実装 |
| **Cloud SQL for PostgreSQL 16** | プライマリ DB | requirements.md Section 6 |
| **Pub/Sub** | 非同期ジョブキュー(Core API → AI Agent) | AD4=C, AD5=A |
| **Secret Manager** | シークレット一元管理 | NR-A7=A, NFR-S-02 |
| **Cloud Storage** | AI Agent の中間生成物保管、Terraform state backend | 必要用途のみ |
| **Cloud Logging** | 構造化ログ集約 | NFR-M-02 |
| **Cloud Monitoring** | メトリクス / ダッシュボード / アラート | NFR-M-03 |
| **Cloud IAM** | サービスアカウント・ロール管理 | NR-A8=A |
| **VPC** + **Serverless VPC Access Connector** | ネットワーク分離 | NR-A6=A, NFR-S-04 |
| **Cloud Scheduler**(将来) | リマインダー、定期タスク | Phase2 |
| **Cloud Build**(未使用) | ビルドは GitHub Actions(U-H)で実施 | CI は U-H 管轄 |

### 1.2 Vercel(フロントエンド側)

| 項目 | 選定 |
|---|---|
| ホスティング | Vercel(Hobby プランで開始、必要に応じて Pro 昇格) |
| デプロイ連携 | GitHub Actions から Vercel CLI(U-H で実装) |
| プレビューデプロイ | PR 自動プレビュー(Vercel の標準機能) |
| 環境変数 | Vercel プロジェクト設定で dev / preview / prod の 3 環境に対応 |
| カスタムドメイン | 未定(MVP は Vercel 提供のサブドメイン) |

---

## 2. Terraform 構成

### 2.1 バージョンとプロバイダ

| 項目 | 選定 |
|---|---|
| Terraform 本体 | `>= 1.9, < 2.0` |
| プロバイダ | `hashicorp/google ~> 5.0`, `hashicorp/google-beta ~> 5.0`, `vercel/vercel ~> 2.0` |
| バージョン管理 | `tfenv` or asdf 使用を推奨(`/infra/.tool-versions`) |

### 2.2 State Backend

- **GCS バケット**: `atm-tf-state`(prod 側 GCP プロジェクトに配置)
- **バケット設定**:
  - ロケーション: `asia-northeast1`
  - バージョニング: 有効
  - ライフサイクル: 90 日以上経過した非最新バージョンを削除
  - 暗号化: Google-managed encryption key(デフォルト)
  - アクセス: Terraform 実行 SA のみ許可
- **State ロック**: GCS は自動ロック対応
- **workspace 分離**: 環境ごとに `/infra/envs/{dev,prod}/` のディレクトリ別で state を分離(workspace 機能は使用しない)

### 2.3 モジュール構成(プレビュー、詳細は Infrastructure Design で確定)

```
/infra/
├── modules/
│   ├── networking/           # VPC、Subnet、VPC Access Connector
│   ├── cloud-sql/            # Cloud SQL インスタンス、DB、ユーザー
│   ├── cloud-run/            # Cloud Run サービス(再利用可能モジュール)
│   ├── cloud-run-jobs/       # サンドボックス用
│   ├── pubsub/               # トピック、サブスクリプション
│   ├── secret-manager/       # シークレット定義と IAM
│   ├── iam/                  # サービスアカウント、ロール
│   ├── monitoring/           # ダッシュボード、アラートポリシー
│   ├── storage/              # Cloud Storage バケット
│   └── vercel/               # Vercel プロジェクト
├── envs/
│   ├── dev/
│   │   ├── main.tf, variables.tf, outputs.tf
│   │   └── terraform.tfvars(.gitignore)
│   └── prod/
│       └── ...
├── shared/                   # 共通 locals、data sources
└── .tool-versions
```

---

## 3. GCP プロジェクト構成

### 3.1 プロジェクト分離(NR-A1=A に基づく dev + prod)

| プロジェクト ID | 環境 | 請求アカウント | 用途 |
|---|---|---|---|
| `atm-dev` | dev | 個人 / 同一請求アカウント | 開発・検証 |
| `atm-prod` | prod | 同上 | 本番 |
| `atm-shared`(任意) | shared | 同上 | Terraform state、Artifact Registry 共有(MVP では省略、後で昇格可) |

**注**: MVP は `atm-shared` を作らず、Terraform state は `atm-prod` の GCS に配置。

### 3.2 リージョン

- **プライマリリージョン**: `asia-northeast1`(東京)
- **理由**: ユーザー(日本)からのレイテンシ最小化、Cloud SQL / Cloud Run / Pub/Sub すべて対応済み
- **マルチリージョン**: MVP では採用せず(コスト上昇)、Phase2 で検討

---

## 4. CI/CD インテグレーション(U-H で実装)

U-A は以下の前提条件のみ提供し、実装は U-H:

- Terraform 実行用 SA(`atm-terraform@atm-prod.iam.gserviceaccount.com`)の発行
- GitHub Actions が使う Workload Identity Federation の設定(`atm-ci-deploy@` SA)
- Terraform plan/apply の PR ワークフロー(U-H の `infra.yml`)
- 手動承認ゲート(prod apply 時)

---

## 5. 主要な技術選定の根拠サマリ

| 判断 | 採用 | 代替案 | 根拠 |
|---|---|---|---|
| コンテナ実行 | Cloud Run(1st/2nd gen) | GKE | サーバレス管理不要、コスト低、MVP 規模に適 |
| DB | Cloud SQL PostgreSQL | Firestore / AlloyDB / 自前 Postgres on GCE | リレーショナル必要、AlloyDB は高価、自前運用は過剰 |
| 非同期キュー | Pub/Sub | Cloud Tasks | 本用途(fanout 想定無し)なら Cloud Tasks でも可だが、Pub/Sub の方が柔軟 |
| 秘密管理 | Secret Manager | KMS 単独、HashiCorp Vault | GCP ネイティブで最小構成 |
| ネットワーク | VPC + Serverless VPC Access Connector | Direct VPC Egress(2nd gen) | **Phase2 で Direct VPC Egress に移行検討**(コスト最適化) |
| 監視 | Cloud Monitoring + Cloud Logging | Datadog、New Relic | GCP 内で完結、無料枠大きい |
| FE ホスティング | Vercel | Cloudflare Pages、Netlify、Cloud Run on GCP | Next.js との親和性最高、プレビューデプロイが優秀 |
| IaC | Terraform | Pulumi、GCP Config Connector | 最も成熟、GitHub Actions 連携豊富 |

---

## 6. MVP 以降の技術判断(参考)

Phase2 以降で検討する技術変更:

- **Direct VPC Egress**(Cloud Run 2nd gen)への移行 → VPC Connector 廃止で $8〜15/月削減
- **pgbouncer** 導入 → Cloud SQL 接続プール飽和の防止
- **Cloud SQL Read Replica** → 読み取り負荷の水平分散
- **Cloud CDN** → 静的アセットの配信最適化(Vercel 側で既に解決している可能性あり)
- **Workload Identity Federation** → GitHub Actions の長期 SA キー廃止(U-A からは MVP でも推奨)
- **BigQuery** → 監査ログ・メトリクスの長期分析
