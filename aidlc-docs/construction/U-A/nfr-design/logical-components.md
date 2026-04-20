# U-A Infrastructure — 論理コンポーネント (Logical Components)

**ユニット**: U-A Infrastructure
**作成日**: 2026-04-19
**前提**: [nfr-design-patterns.md](./nfr-design-patterns.md)

各 GCP リソース / Vercel リソースを **論理コンポーネント** として列挙し、配置モジュール(shared / service)とファイル(`*.tf`)を明示する。Infrastructure Design の叩き台とする。

---

## 1. shared モジュール内の論理コンポーネント

### 1.1 ネットワーク(`modules/shared/network.tf`)

| ID | リソース | 設定ハイライト |
|---|---|---|
| NW-1 | `google_compute_network` `atm-vpc` | `auto_create_subnetworks = false` |
| NW-2 | `google_compute_subnetwork` `atm-subnet-private` | region=asia-northeast1、CIDR=`10.10.0.0/24`、`private_ip_google_access = true` |
| NW-3 | `google_vpc_access_connector` `atm-connector` | machine_type=`e2-micro`、min=2、max=3 |
| NW-4 | `google_compute_global_address` `atm-psa-range` | Private Service Access 用 IP レンジ(Cloud SQL Private IP) |
| NW-5 | `google_service_networking_connection` `atm-psa` | VPC と Private Service Access の接続 |
| NW-6 | `google_compute_firewall` `atm-deny-all-ingress` | VPC 内部のデフォルト deny |
| NW-7 | `google_compute_firewall` `atm-allow-connector-to-sql` | VPC Connector → Cloud SQL のみ許可 |

### 1.2 データベース(`modules/shared/database.tf`)

| ID | リソース | 備考 |
|---|---|---|
| DB-1 | `google_sql_database_instance` `atm-main` | PostgreSQL 16, db-f1-micro, Private IP, ZONAL, PITR 7 日 |

**outputs**: `instance_name`, `connection_name`, `private_ip_address`

### 1.3 共通シークレット(Type B, `modules/shared/secrets.tf`)

| ID | リソース | 用途 |
|---|---|---|
| SEC-1 | `google_secret_manager_secret` `github-app-private-key` | GitHub App の秘密鍵(pem ファイル) |
| SEC-2 | `google_secret_manager_secret` `github-webhook-secret` | Webhook HMAC 検証用シークレット |
| SEC-3 | `google_secret_manager_secret` `anthropic-api-key` | Claude API キー |
| SEC-4 | `google_secret_manager_secret` `firebase-admin-key`(Phase2) | Firebase Auth 管理者鍵 |

※ 値は Terraform で管理せず、手動で `gcloud secrets versions add` or UI から投入。

**outputs**: `secrets = { github_app_private_key = ..., github_webhook_secret = ..., anthropic_api_key = ... }`

### 1.4 メッセージング(`modules/shared/messaging.tf`)

| ID | リソース | 用途 |
|---|---|---|
| MSG-1 | `google_pubsub_topic` `ai-jobs` | Core API から AI Agent へのジョブ投入 |
| MSG-2 | `google_pubsub_topic` `ai-jobs-dlq` | ai-jobs の dead letter |
| MSG-3 | `google_pubsub_topic` `github-webhook-events` | Webhook 受信後の内部イベント(必要に応じて) |

**outputs**: `topics = { ai_jobs = ..., ai_jobs_dlq = ..., github_webhook_events = ... }`

### 1.5 ストレージ(`modules/shared/storage.tf`)

| ID | リソース | 用途 |
|---|---|---|
| ST-1 | `google_storage_bucket` `atm-sandbox-artifacts-${env}` | AI Agent の中間生成物、コード分析キャッシュ |

**bucket 設定**:
- uniform_bucket_level_access = true
- public_access_prevention = "enforced"
- lifecycle: 30 日後に自動削除
- versioning: 無効(一時データのため)

**outputs**: `storage = { sandbox_bucket = ..., sandbox_bucket_url = ... }`

### 1.6 IAM / CI(`modules/shared/iam.tf`)

| ID | リソース | 用途 |
|---|---|---|
| IAM-1 | `google_iam_workload_identity_pool` `github-actions` | GitHub Actions OIDC Pool |
| IAM-2 | `google_iam_workload_identity_pool_provider` `github` | GitHub Actions Provider(attribute condition で特定リポ限定) |
| IAM-3 | `google_service_account` `atm-terraform` | Terraform 実行用 SA |
| IAM-4 | `google_project_iam_member` or Custom Role | `atm-terraform` への最小化 admin 権限 |
| IAM-5 | `google_service_account_iam_member` | `atm-terraform` への WIF binding(`attribute.workflow/infra`) |
| IAM-6 | `google_service_account` `atm-monitoring`(任意) | 監視閲覧専用 SA |

**outputs**: `wif = { pool_name = ..., provider_name = ... }`、`iam = { terraform_sa_email = ... }`

### 1.7 オブザーバビリティ(`modules/shared/observability.tf`)

| ID | リソース | 用途 |
|---|---|---|
| OB-1 | `google_monitoring_notification_channel` `email-primary` | アラート通知チャンネル(var.alert_notification_email) |
| OB-2 | `google_monitoring_dashboard` `atm-overview` | プロジェクト全体の健全性ダッシュボード |
| OB-3 | `google_monitoring_dashboard` `atm-database` | Cloud SQL 用ダッシュボード |
| OB-4 | `google_monitoring_alert_policy` `atm-sql-connection-exhaustion` | DB 接続数 80% アラート |
| OB-5 | `google_billing_budget` `atm-monthly-budget` | $30 警告 / $50 超過 |

**outputs**: `observability = { notification_channel_id = ... }`

---

## 2. service モジュール内の論理コンポーネント(例: core-api)

### 2.1 core-api(`modules/core-api/`)

| ID | ファイル | リソース | 備考 |
|---|---|---|---|
| CA-1 | `iam.tf` | `google_service_account` `atm-core-api` | Cloud Run ランタイム SA |
| CA-2 | `iam.tf` | `google_service_account` `atm-deploy-core-api` | CI デプロイ SA |
| CA-3 | `iam.tf` | `google_cloud_run_v2_service_iam_member` | deploy SA に `roles/run.developer` |
| CA-4 | `iam.tf` | `google_service_account_iam_member` | deploy SA → runtime SA の `actAs` |
| CA-5 | `iam.tf` | `google_service_account_iam_member` | deploy SA への WIF binding(attribute.workflow/core-api) |
| CA-6 | `iam.tf` | `google_project_iam_member` | runtime SA に `roles/cloudsql.client`, `roles/pubsub.publisher` 等 |
| CA-7 | `database.tf` | `google_sql_database` `atm_core_api` | 論理 DB |
| CA-8 | `database.tf` | `google_sql_user` `core_api` + `random_password` | DB ユーザー(パスワードは Secret 化) |
| CA-9 | `secrets.tf` | `google_secret_manager_secret` `core-api-db-password`(Type A) | + version、runtime SA に accessor |
| CA-10 | `secrets.tf` | `google_secret_manager_secret` `core-api-session-key`(Type A) | セッション署名鍵 |
| CA-11 | `secrets.tf` | `google_secret_manager_secret_iam_member` x N(Type B 消費) | shared secret への accessor(for_each) |
| CA-12 | `cloud-run.tf` | `google_cloud_run_v2_service` `core-api` | INGRESS_ALL, concurrency=80, max=10, min=0, CPU boost, VPC Connector, env secrets |
| CA-13 | `messaging.tf` | `google_pubsub_topic_iam_member` | runtime SA が `ai-jobs` に publisher |
| CA-14 | `monitoring.tf` | `google_monitoring_alert_policy` `atm-core-api-error-rate-high` | |
| CA-15 | `monitoring.tf` | `google_monitoring_alert_policy` `atm-core-api-latency-high` | |

### 2.2 ai-agent(`modules/ai-agent/`)

| ID | ファイル | リソース | 備考 |
|---|---|---|---|
| AA-1 | `iam.tf` | `google_service_account` `atm-ai-agent` | ランタイム |
| AA-2 | `iam.tf` | `google_service_account` `atm-deploy-ai-agent` | デプロイ |
| AA-3〜5 | `iam.tf` | deploy SA の各 binding(Cloud Run + actAs + WIF) | CA-3〜5 と同パターン |
| AA-6 | `iam.tf` | runtime SA の `roles/pubsub.subscriber`, `roles/storage.objectUser` | |
| AA-7 | `secrets.tf` | Type B 消費(anthropic_api_key, github_app_private_key) | IAM binding のみ |
| AA-8 | `cloud-run.tf` | `google_cloud_run_v2_service` `ai-agent` | INGRESS_INTERNAL_ONLY |
| AA-9 | `cloud-run.tf` | `google_cloud_run_v2_job` `ai-agent-sandbox` | Cloud Run Jobs(サンドボックス実行) |
| AA-10 | `messaging.tf` | `google_pubsub_subscription` `ai-jobs-worker` | DLQ 付き、ack_deadline=600s |
| AA-11 | `storage.tf` | `google_storage_bucket_iam_member` | sandbox bucket への `objectUser` |
| AA-12 | `monitoring.tf` | `ai-agent-*` アラート 2 種 | 同パターン |

### 2.3 frontend(`modules/frontend/`)

| ID | ファイル | リソース | 備考 |
|---|---|---|---|
| FE-1 | `vercel.tf` | `vercel_project` `atm-frontend` | framework=nextjs |
| FE-2 | `vercel.tf` | `vercel_project_environment_variable` x N | 環境変数(dev/preview/production ごと) |
| FE-3 | `secrets.tf` | Vercel プロジェクト用の secret 参照 | API URL 等 |
| FE-4 | `dns.tf`(将来) | `vercel_project_domain` | カスタムドメイン、MVP では未作成 |

### 2.4 github-app(`modules/github-app/`)

| ID | ファイル | リソース | 備考 |
|---|---|---|---|
| GA-1 | `secrets.tf` | Type B secrets(github_app_private_key, github_webhook_secret)への binding | core-api runtime SA に accessor 付与 |
| GA-2 | `iam.tf` | (現状なし、必要に応じて追加) | |

※ GitHub App 自体の manifest は `/github-app/manifest.yml`(Terraform 外)で管理。

---

## 3. コンポーネント依存関係(apply 順序)

```
shared.network (NW-*)
     │
     ├─→ shared.database (DB-1, PSA は network に依存)
     │
     ├─→ shared.storage (ST-1)
     ├─→ shared.messaging (MSG-*)
     ├─→ shared.secrets (SEC-*)
     ├─→ shared.iam (IAM-*, WIF は network に非依存)
     │
     └─→ shared.observability (OB-*, notification channel 経由でアラート)

           ↓ すべての shared が完了後

core-api (CA-*) ─┐
ai-agent (AA-*) ─┤── 並列 apply 可能(shared の outputs に依存するが互いに独立)
frontend (FE-*) ─┤
github-app (GA-*)─┘
```

---

## 4. Terraform state に含まれるリソース概算

| モジュール | リソース数概算 |
|---|---|
| shared | 約 35(NW=7, DB=1, SEC=4, MSG=3, ST=1, IAM=6, OB=5 + 関連 binding/version) |
| core-api | 約 25 |
| ai-agent | 約 22 |
| frontend | 約 10 |
| github-app | 約 3 |
| **合計(env あたり)** | **約 95 リソース** |

dev と prod の 2 環境で合計 約 190 リソースを Terraform 管理。

---

## 5. 未解決事項(Infrastructure Design で確定)

上記仮置き値はすべて Infrastructure Design ステージで最終決定:
- NW-2 の CIDR(`10.10.0.0/24`)
- DB-1 の instance name
- ST-1 のバケット命名規則
- Cloud Run CPU/Memory の具体値
- Pub/Sub subscription の ack_deadline / retain_acked_messages 設定
- OB-2/3 ダッシュボードの具体的な widget 定義
