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

**方針**: 各環境の state は**自分の GCP プロジェクト内**に配置する(環境間の完全分離)。

#### 2.2.1 環境ごとの state バケット

| 環境 | GCS バケット名 | 配置先プロジェクト |
|---|---|---|
| dev | `atm-tf-state-dev` | `atm-dev` |
| prod | `atm-tf-state-prod` | `atm-prod` |

**バケット設定(共通)**:
- ロケーション: `asia-northeast1`
- バージョニング: 有効(誤消去・破損時の復旧)
- ライフサイクル: 90 日以上経過した非最新バージョンを削除
- 暗号化: Google-managed encryption key(デフォルト。必要に応じて CMEK に昇格可)
- アクセス: 当該環境の Terraform 実行 SA のみに `roles/storage.objectAdmin` を付与
- uniform bucket-level access: 有効(個別 object IAM を使わない)
- public access prevention: 有効

#### 2.2.2 Backend 宣言

各 env が自分の backend を宣言する:

```hcl
# envs/dev/backend.tf
terraform {
  backend "gcs" {
    bucket = "atm-tf-state-dev"
    prefix = "terraform/state"
  }
}

# envs/prod/backend.tf
terraform {
  backend "gcs" {
    bucket = "atm-tf-state-prod"
    prefix = "terraform/state"
  }
}
```

#### 2.2.3 環境分離の利点

- **Blast radius 限定**: dev プロジェクトでの操作ミス・権限設定の事故が prod state に波及しない
- **IAM 分離**: dev 操作者のロールは `atm-dev` プロジェクト内に閉じ込められる(prod state へはアクセス権がない)
- **課金の明確化**: 各環境の state storage コストは当該プロジェクトの請求に計上
- **メンタルモデル**: 「ある環境のすべては、その環境のプロジェクト内にある」でシンプル

#### 2.2.4 State ロック

- GCS backend は自動でオブジェクトロックに対応(Terraform 0.15+)
- 明示的なロック backend(Consul、Dynamo 等)は不要

#### 2.2.5 Bootstrap(初回セットアップ)

state バケット自身を Terraform で管理できない「卵と鶏」問題の回避:

1. **初回のみ手動でバケット作成**(各環境 1 回のみ、`/infra/bootstrap/` に gcloud コマンド手順書を置く):
   ```bash
   # dev
   gcloud storage buckets create gs://atm-tf-state-dev \
     --project=atm-dev --location=asia-northeast1 \
     --uniform-bucket-level-access --public-access-prevention
   gcloud storage buckets update gs://atm-tf-state-dev \
     --versioning

   # prod も同様
   ```
2. 作成後は **Terraform backend として使用開始**(`terraform init` で初期化)
3. バケット自体の **設定の変更**(ライフサイクル追加など)は Terraform でインポートして以降管理(`terraform import`)

#### 2.2.6 Workspace について

- **Terraform workspaces は使用しない**
- 環境分離は `/infra/envs/{dev,prod}/` のディレクトリ別で実現(明示的かつ設定の分岐が可視化される)

### 2.3 モジュール構成(確定)

#### 2.3.1 設計原則

- **Service vertical slice**: サービスごとに 1 モジュール(`core-api`, `ai-agent`, `frontend`, `github-app`)。そのサービスが必要とするすべてのインフラ(Cloud Run、SA、IAM、固有 secret、Pub/Sub subscription、アラート)を 1 モジュール内に集約
- **リソース単位のモジュール化は行わない**(ラッパーのみで抽象化価値がないアンチパターンを回避)
- **共通基盤は単一 `shared` モジュール**: 横断的なインフラ(VPC、Cloud SQL instance、Type B shared secrets、Pub/Sub topics、通知チャンネル等)を 1 モジュールに集約、ファイル分割で整理
- **ファイル命名規則を shared / service で統一**: 関心ごとに `.tf` ファイルを分け、モジュールのルートを `ls` するだけで何があるか分かる状態を維持
- **1 state per env**(MVP): `envs/{dev,prod}/` ごとに単一 state。Terraform 依存解決に順序を任せる。規模拡大時は state 分割に昇格可能

#### 2.3.2 ディレクトリ構造

```
/infra/
├── modules/
│   ├── shared/                    # 共通基盤(単一モジュール、ファイル分割で整理)
│   │   ├── main.tf                # locals, providers 要件
│   │   ├── variables.tf
│   │   ├── network.tf             # VPC, Subnet, VPC Access Connector, Firewall
│   │   ├── database.tf            # Cloud SQL instance, 自動バックアップ, Private Service Access
│   │   ├── secrets.tf             # Type B shared secrets(GitHub App key, Anthropic key, Firebase admin key 等)
│   │   ├── messaging.tf           # Pub/Sub topics(shared)+ DLQ topics
│   │   ├── storage.tf             # Cloud Storage bucket(サンドボックス中間生成物用)
│   │   ├── iam.tf                 # Workload Identity Federation, 横断 SA
│   │   ├── observability.tf       # Notification channels, プロジェクト全体ダッシュボード
│   │   └── outputs.tf
│   │
│   ├── core-api/                  # サービス単位(Cloud Run / Go)
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── iam.tf                 # SA + project/shared IAM binding
│   │   ├── cloud-run.tf           # Cloud Run サービス本体
│   │   ├── database.tf            # DB user, password, 論理 DB
│   │   ├── secrets.tf             # 固有 secret(session_signing_key 等)+ shared secret IAM binding
│   │   ├── messaging.tf           # Pub/Sub subscriptions(このサービスが購読/発行)
│   │   ├── monitoring.tf          # サービス固有アラート・ダッシュボード
│   │   └── outputs.tf
│   │
│   ├── ai-agent/                  # サービス単位(Cloud Run + Cloud Run Jobs / Python)
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── iam.tf
│   │   ├── cloud-run.tf           # Cloud Run service + Cloud Run Jobs(sandbox)
│   │   ├── secrets.tf             # 固有 + shared IAM binding
│   │   ├── messaging.tf           # Pub/Sub subscriptions
│   │   ├── storage.tf             # Cloud Storage bucket access binding
│   │   ├── monitoring.tf
│   │   └── outputs.tf
│   │
│   ├── frontend/                  # Vercel project
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── vercel.tf              # Vercel project 定義
│   │   ├── secrets.tf             # Vercel 環境変数 & secrets
│   │   ├── dns.tf                 # (将来)カスタムドメイン
│   │   └── outputs.tf
│   │
│   └── github-app/                # GitHub App の GCP 側受け皿
│       ├── main.tf
│       ├── variables.tf
│       ├── iam.tf                 # Webhook 受信関連の IAM(Core API SA への bindings 含む)
│       ├── secrets.tf             # GitHub App 関連 secret の IAM binding(定義は shared 側)
│       └── outputs.tf
│
└── envs/
    ├── dev/
    │   ├── backend.tf             # GCS state backend
    │   ├── providers.tf           # google, google-beta, vercel providers
    │   ├── variables.tf
    │   ├── terraform.tfvars       # (.gitignore で除外、値は手動管理)
    │   ├── main.tf                # shared → services のモジュール組立
    │   └── outputs.tf
    └── prod/
        └── ...(dev と同構成)
```

#### 2.3.3 ファイル命名規則(全モジュール共通)

| ファイル名 | 責務 | 配置 |
|---|---|---|
| `main.tf` | locals、バージョン制約 | 全モジュール |
| `variables.tf` | 入力変数(description に用途を記載) | 全モジュール |
| `outputs.tf` | 他モジュールへ公開する値 | 全モジュール |
| `network.tf` | VPC、Subnet、Connector、Firewall | shared のみ |
| `database.tf` | Cloud SQL instance(shared)/ DB user・論理 DB(service) | shared + service |
| `secrets.tf` | Secret Manager 定義 + IAM binding + Cloud Run env 参照 | shared + service |
| `messaging.tf` | Pub/Sub topic(shared)/ subscription(service) | shared + service |
| `storage.tf` | Cloud Storage bucket + access binding | shared + service |
| `iam.tf` | SA 作成、ロール付与、Workload Identity | 全モジュール |
| `cloud-run.tf` | Cloud Run service + Jobs | service のみ |
| `monitoring.tf` | ダッシュボード、アラート | shared + service |
| `vercel.tf` | Vercel project | frontend のみ |
| `dns.tf` / `webhook.tf` | 用途特化(必要時のみ) | service のみ |

#### 2.3.4 責務の分担ルール

| 関心 | shared で定義 | service で定義 |
|---|---|---|
| VPC / Subnet / VPC Access Connector / Firewall | ✅ | — |
| Cloud SQL インスタンス + 自動バックアップ | ✅ | — |
| DB 論理 DB + ユーザー + パスワード | — | ✅ 各サービスが自分用を宣言(orphan 防止、機密の service 閉じ込め) |
| **Type A 固有 secret**(DB パスワード、セッション署名鍵 等) | — | ✅ service で定義・消費 |
| **Type B 共通 secret**(GitHub App key、Anthropic key、Firebase admin key 等、外部アイデンティティ) | ✅ 定義 | ✅ IAM binding のみ(消費宣言) |
| Pub/Sub topic + DLQ | ✅ | — |
| Pub/Sub subscription(購読側に所有権) | — | ✅ |
| SA 作成 + ロール付与(サービスランタイム用) | — | ✅ |
| SA 作成 + ロール付与(横断用、Terraform 実行 SA など) | ✅ | — |
| **Workload Identity Federation Pool + Provider**(GitHub Actions 用、全サービスで共有) | ✅ | — |
| **Terraform 実行 SA + WIF binding(infra.yml 用)** | ✅ | — |
| **サービスデプロイ SA + WIF binding**(例: `atm-deploy-core-api`、`run.developer` ロール) | — | ✅ 各サービスが自分用を宣言 |
| Cloud Run service + Jobs | — | ✅ |
| 通知チャンネル(Notification Channel) | ✅ | — |
| プロジェクト全体ダッシュボード | ✅ | — |
| サービス固有アラート・ダッシュボード | — | ✅ |
| Cloud Storage bucket | ✅(共有 bucket) | IAM binding のみ |

#### 2.3.5 Secret 消費の明示化パターン

サービスモジュールの `variables.tf` で **消費する shared secret を型で明示**:

```hcl
# modules/core-api/variables.tf(抜粋)
variable "shared_secrets" {
  description = "Secrets consumed by core-api from shared module"
  type = object({
    github_app_private_key = string  # GitHub App JWT 生成
    github_webhook_secret  = string  # Webhook HMAC 検証
    # anthropic_api_key や firebase_admin_key は core-api では使わないため含めない
  })
}
```

```hcl
# modules/core-api/secrets.tf(抜粋)
# Type A: このサービス固有の secret(定義 + 消費)
resource "random_password" "session_key" { length = 64 }

resource "google_secret_manager_secret" "session_key" {
  secret_id = "core-api-session-key"
  replication { auto {} }
}

resource "google_secret_manager_secret_version" "session_key_v1" {
  secret      = google_secret_manager_secret.session_key.id
  secret_data = random_password.session_key.result
}

resource "google_secret_manager_secret_iam_member" "session_key_accessor" {
  secret_id = google_secret_manager_secret.session_key.id
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:${google_service_account.core_api.email}"
}

# Type B: shared で定義された共通 secret への IAM binding のみ
resource "google_secret_manager_secret_iam_member" "shared_accessors" {
  for_each  = var.shared_secrets
  secret_id = each.value
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:${google_service_account.core_api.email}"
}
```

#### 2.3.6 envs の composition(apply 順序と配線)

```hcl
# envs/dev/main.tf

module "shared" {
  source     = "../../modules/shared"
  project_id = var.project_id
  region     = var.region
}

module "core_api" {
  source     = "../../modules/core-api"
  project_id = var.project_id
  region     = var.region

  network           = module.shared.network            # { vpc_id, connector_id, ... }
  database_instance = module.shared.database_instance  # { name, connection_name, private_ip }

  shared_secrets = {
    github_app_private_key = module.shared.secrets.github_app_private_key
    github_webhook_secret  = module.shared.secrets.github_webhook_secret
  }

  shared_topics = module.shared.messaging.topics       # 発行先 topic
}

module "ai_agent" {
  source     = "../../modules/ai-agent"
  project_id = var.project_id
  region     = var.region

  network               = module.shared.network
  sandbox_storage_bucket = module.shared.storage.sandbox_bucket

  shared_secrets = {
    anthropic_api_key      = module.shared.secrets.anthropic_api_key
    github_app_private_key = module.shared.secrets.github_app_private_key
  }

  shared_topics = module.shared.messaging.topics
}

module "frontend" {
  source = "../../modules/frontend"
  # Vercel は GCP リソースと独立、shared への依存なし
}

module "github_app" {
  source     = "../../modules/github-app"
  project_id = var.project_id

  core_api_service_account_email = module.core_api.service_account_email
}
```

依存方向は常に **`shared` → `services`** の一方向。services 間の直接依存は **`github-app` → `core-api` SA 参照のみ** に限定。

#### 2.3.7 CI/CD 権限の配置(選択肢 A: 共有 WIF + サービス別 deploy SA)

GitHub Actions からの deploy は **Workload Identity Federation**(WIF)で行い、SA キー JSON は発行しない(NDB-1 推奨通り)。

**shared に置くもの**:

```hcl
# modules/shared/iam.tf(抜粋)

# GitHub Actions からの OIDC 連携用 Pool
resource "google_iam_workload_identity_pool" "github" {
  workload_identity_pool_id = "github-actions"
  display_name              = "GitHub Actions Pool"
}

# Provider(特定リポジトリからのみ受け入れ)
resource "google_iam_workload_identity_pool_provider" "github" {
  workload_identity_pool_id          = google_iam_workload_identity_pool.github.workload_identity_pool_id
  workload_identity_pool_provider_id = "github"

  attribute_condition = "assertion.repository == 'soneda-yuya/ai-task-management'"
  attribute_mapping = {
    "google.subject"       = "assertion.sub"
    "attribute.repository" = "assertion.repository"
    "attribute.workflow"   = "assertion.workflow"
    "attribute.ref"        = "assertion.ref"
  }
  oidc {
    issuer_uri = "https://token.actions.githubusercontent.com"
  }
}

# Terraform 実行 SA(infra.yml 専用)
resource "google_service_account" "terraform" {
  account_id   = "atm-terraform"
  display_name = "Terraform runner (infra.yml)"
}

resource "google_project_iam_member" "terraform_editor" {
  project = var.project_id
  role    = "roles/editor"  # 実際は Custom Role で最小化
  member  = "serviceAccount:${google_service_account.terraform.email}"
}

# infra.yml workflow からのみ impersonate 可能
resource "google_service_account_iam_member" "terraform_wif" {
  service_account_id = google_service_account.terraform.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/attribute.workflow/infra"
}
```

**各 service に置くもの**(例: core-api):

```hcl
# modules/core-api/iam.tf(抜粋)

# Core API デプロイ専用 SA
resource "google_service_account" "deploy" {
  account_id   = "atm-deploy-core-api"
  display_name = "Core API deploy (core-api.yml)"
}

# Cloud Run リビジョンを更新できる最小権限
resource "google_cloud_run_v2_service_iam_member" "deploy_developer" {
  project  = var.project_id
  location = var.region
  name     = google_cloud_run_v2_service.core_api.name
  role     = "roles/run.developer"
  member   = "serviceAccount:${google_service_account.deploy.email}"
}

# ランタイム SA を impersonate する権限(deploy 時に actAs が必要)
resource "google_service_account_iam_member" "deploy_act_as_runtime" {
  service_account_id = google_service_account.core_api.name  # ランタイム SA
  role               = "roles/iam.serviceAccountUser"
  member             = "serviceAccount:${google_service_account.deploy.email}"
}

# core-api.yml workflow からのみ impersonate 可能
resource "google_service_account_iam_member" "deploy_wif" {
  service_account_id = google_service_account.deploy.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "principalSet://iam.googleapis.com/${var.wif_pool_name}/attribute.workflow/core-api"
}
```

**SA 一覧と用途**:

| SA 名 | 配置モジュール | 用途 | WIF binding(workflow 属性) |
|---|---|---|---|
| `atm-terraform@` | shared | Terraform plan/apply 実行 | `infra` |
| `atm-core-api@` | core-api | Cloud Run ランタイム(コード実行) | — |
| `atm-deploy-core-api@` | core-api | Cloud Run リビジョン更新デプロイ | `core-api` |
| `atm-ai-agent@` | ai-agent | Cloud Run / Jobs ランタイム | — |
| `atm-deploy-ai-agent@` | ai-agent | Cloud Run デプロイ | `ai-agent` |
| `atm-monitoring@`(任意) | shared | 監視ダッシュボード閲覧 | — |

**frontend(Vercel)**: GCP SA 不要。Vercel のデプロイは Vercel の Git 連携で行うか、GitHub Actions から Vercel CLI を使う場合は Vercel Access Token を GitHub Secrets に保存(これは U-H で設定、U-A の範疇外)。

**github-app**: デプロイ対象コードがないため CI 用 SA 不要。

---

## 3. GCP プロジェクト構成

### 3.1 プロジェクト分離(NR-A1=A に基づく dev + prod)

| プロジェクト ID | 環境 | 請求アカウント | 用途 |
|---|---|---|---|
| `atm-dev` | dev | 個人 / 同一請求アカウント | 開発・検証、dev の state バケットも同プロジェクト内 |
| `atm-prod` | prod | 同上 | 本番、prod の state バケットも同プロジェクト内 |

**state 配置方針**: 各環境の Terraform state は**自プロジェクト内**の GCS バケットに配置(Section 2.2 参照)。`atm-shared` のような共有プロジェクトは MVP では作らない。

### 3.2 リージョン

- **プライマリリージョン**: `asia-northeast1`(東京)
- **理由**: ユーザー(日本)からのレイテンシ最小化、Cloud SQL / Cloud Run / Pub/Sub すべて対応済み
- **マルチリージョン**: MVP では採用せず(コスト上昇)、Phase2 で検討

---

## 4. CI/CD インテグレーション(U-A 提供 / U-H 実装)

### 4.1 U-A で発行するリソース(詳細は Section 2.3.7 参照)

- **Workload Identity Pool**(`github-actions`)と Provider(issuer: `token.actions.githubusercontent.com`、`soneda-yuya/ai-task-management` リポ限定)— shared モジュール
- **Terraform 実行 SA** (`atm-terraform@`) + WIF binding(workflow=infra)— shared モジュール
- **サービスデプロイ SA** (`atm-deploy-core-api@`, `atm-deploy-ai-agent@`)+ Cloud Run 権限 + WIF binding — 各 service モジュール

### 4.2 U-H で実装するもの

- `.github/workflows/infra.yml`(Terraform plan on PR / apply on main)
- `.github/workflows/core-api.yml`(build + Cloud Run deploy)
- `.github/workflows/ai-agent.yml`(build + Cloud Run deploy)
- `.github/workflows/frontend.yml`(build + Vercel deploy)
- `.github/workflows/proto.yml`(Buf lint / generate 差分検出)
- prod apply 時の手動承認ゲート(GitHub Environments 機能)
- Branch protection ルール、Required reviews
- Vercel Access Token の GitHub Secrets 設定

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
| モジュール粒度 | サービス垂直スライス + 単一 shared | リソース単位モジュール、Terragrunt | リソース単位は抽象化価値なしのアンチパターン。Terragrunt は MVP 規模では過剰 |
| Secret 分類 | Type A(service 固有)+ Type B(shared 外部アイデンティティ) | 全 service で重複定義、全 shared 集約 | 重複は同期問題、全集約は vertical slice 崩壊 |
| State 戦略 | 1 state per env(MVP) | state 分割、workspaces | MVP 規模ではシンプルさ優先、規模拡大時に分割昇格可 |

---

## 6. MVP 以降の技術判断(参考)

Phase2 以降で検討する技術変更:

- **Direct VPC Egress**(Cloud Run 2nd gen)への移行 → VPC Connector 廃止で $8〜15/月削減
- **pgbouncer** 導入 → Cloud SQL 接続プール飽和の防止
- **Cloud SQL Read Replica** → 読み取り負荷の水平分散
- **Cloud CDN** → 静的アセットの配信最適化(Vercel 側で既に解決している可能性あり)
- **Workload Identity Federation** → GitHub Actions の長期 SA キー廃止(U-A からは MVP でも推奨)
- **BigQuery** → 監査ログ・メトリクスの長期分析
