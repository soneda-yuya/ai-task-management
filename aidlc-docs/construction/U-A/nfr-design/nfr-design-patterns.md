# U-A Infrastructure — NFR 設計パターン (NFR Design Patterns)

**ユニット**: U-A Infrastructure
**作成日**: 2026-04-19
**前提**: [nfr-requirements.md](../nfr-requirements/nfr-requirements.md), [tech-stack-decisions.md](../nfr-requirements/tech-stack-decisions.md) 承認済
**回答結果**: NDA-1=A, NDA-2=A, NDB-1=A, NDB-2=A, NDC-1=A(メール値: `soneda.yuya@gmail.com`、Terraform では `var.alert_notification_email` で受け取り、dev/prod の tfvars で指定), NDC-2=A

---

## 1. 可用性・レジリエンスパターン

### P-AV-1: Cloud Run マルチゾーン自動配置(NDA-1=A, UA-N-A-01)
**パターン**: 単一リージョン `asia-northeast1` 内で Cloud Run が自動的に複数ゾーンへリビジョンを配置。
**Terraform 設定**:
- `ingress = "INGRESS_TRAFFIC_ALL"`(Core API)/ `"INGRESS_TRAFFIC_INTERNAL_LOAD_BALANCER"` or `"INGRESS_TRAFFIC_INTERNAL_ONLY"`(AI Agent)
- `scaling { min_instance_count = 0; max_instance_count = 10 }`
- 明示的なゾーン指定はせず、Cloud Run のマネージドプレースメントに任せる

### P-AV-2: Cloud SQL ZONAL availability(NDA-1=A, 予算優先)
**パターン**: ZONAL(単一ゾーン)で開始、MVP 以降で REGIONAL に昇格可能。
**Terraform 設定**:
```hcl
resource "google_sql_database_instance" "main" {
  database_version = "POSTGRES_16"
  settings {
    tier              = "db-f1-micro"
    availability_type = "ZONAL"  # MVP
    # Phase2: "REGIONAL"

    backup_configuration {
      enabled                        = true
      point_in_time_recovery_enabled = true
      start_time                     = "03:00"  # JST 12:00 = UTC 03:00
      backup_retention_settings {
        retained_backups = 7
        retention_unit   = "COUNT"
      }
    }

    ip_configuration {
      ipv4_enabled    = false
      private_network = var.network.vpc_id
    }
  }
  deletion_protection = true  # prod のみ
}
```

### P-AV-3: バックアップ/リストア手順(UA-N-A-03, A-04, A-05)
**RPO 5 分 / RTO 1 時間**(手動リストア):
1. 自動バックアップ毎日 1 回(UTC 03:00 固定)
2. PITR 有効で 7 日以内の任意時点に復旧可能
3. リストア手順書: `/docs/runbooks/db-restore.md`(Code Generation 時に生成)
4. dev 環境で月次リストア訓練(運用開始後)

### P-AV-4: Cloud Run 自動ロールバック(UA-N-R-01)
**パターン**: デプロイ後にヘルスチェックが失敗した場合、前リビジョンへトラフィック戻し。
**Terraform 設定**:
- `traffic { percent = 100; type = "TRAFFIC_TARGET_ALLOCATION_TYPE_LATEST" }`(新リビジョンに 100% 寄せる、失敗時は手動ロールバック)
- または Canary デプロイ:`50/50` → 監視 → 100% 昇格(Phase2 で採用検討)

---

## 2. スケーラビリティパターン

### P-SC-1: Cloud Run 水平スケーリング(NFR-P-05, UA-N-P-03)
**パターン**: concurrency ベースのオートスケーリング。
- `concurrency = 80`(1 インスタンスあたり同時 80 リクエストまで)
- `max_instance_count = 10`(MVP 上限)
- 理論値: 最大 800 同時リクエスト(MVP 規模十分)

### P-SC-2: Pub/Sub 自動スケーリング(UA-N-P-03)
- Pub/Sub は **GCP マネージドで自動スケール**(ユーザー側設定不要)
- Pull subscription の消費者(AI Agent)は Cloud Run の auto-scale と連動

### P-SC-3: Cloud SQL 接続プール戦略(UA-N-P-02)
**課題**: db-f1-micro の `max_connections ≒ 25` × Cloud Run max=10 × 各アプリの接続プール数で超過リスク。
**MVP 対策**:
- 各サービスのアプリ内接続プール上限を 2〜4 に制限(Go: pgxpool, Python: asyncpg pool)
- Cloud SQL `max_connections` を 50 に引き上げ(Terraform で `database_flags`)
- アラート: 接続数 > 80% で通知

**Phase2**:
- pgbouncer サイドカー導入、または Cloud SQL インスタンス昇格

---

## 3. パフォーマンスパターン

### P-PF-1: コールドスタート緩和(NDA-2=A, UA-N-P-01)
**パターン**: 軽量化 + startup CPU boost、min=0 維持でコスト抑制。
**Terraform 設定**:
```hcl
resource "google_cloud_run_v2_service" "this" {
  template {
    containers {
      startup_cpu_boost = true   # ★ コールドスタート時の CPU 倍増
      resources {
        limits = {
          cpu    = "1"
          memory = "512Mi"
        }
        cpu_idle         = true  # アイドル時 CPU 割当なし(コスト削減)
      }
    }
  }
}
```
**アプリ側対策**:
- Go: 静的リンクバイナリ、distroless コンテナ
- Python: AI Agent は依存が重いが、`uv` による高速 import、事前コンパイル済 wheel

### P-PF-2: ネットワーク近接性
- すべての GCP リソースは `asia-northeast1` に配置
- Cloud SQL Private IP アクセスで低レイテンシ(< 1ms RTT)

### P-PF-3: Cloud Run リソース配分(サービスごと、NFR Design ではプレースホルダ)

| サービス | CPU | Memory | 備考 |
|---|---|---|---|
| Core API | 1 vCPU | 512 MiB | 一般 RPC |
| AI Agent Service | 2 vCPU | 1 GiB | LLM 呼び出し + ツール実行 |
| AI Agent Jobs(sandbox) | 2 vCPU | 2 GiB | 実装時のファイル操作用 |

※ 実際の値は Code Generation 時に調整可能。

---

## 4. セキュリティパターン

### P-SE-1: VPC 分離と Private IP(UA-N-S-01, NDB-2)
**Defense in depth**:
- Cloud SQL は **Public IP 無効化**、Private IP のみ
- Cloud Run から VPC Connector 経由のみ接続可能
- Firewall: VPC 外からの inbound はデフォルト deny

### P-SE-2: Workload Identity Federation(NDB-1=A, NFR-S-04)
**パターン**: 長期 SA キー廃止、GitHub OIDC token → 短期 STS token → SA impersonate。
**詳細**: tech-stack-decisions.md Section 2.3.7 を参照。

### P-SE-3: サービスの公開範囲分離(NDB-2=A)
- **Core API**: `INGRESS_TRAFFIC_ALL` + `allow_unauthenticated = true`(アプリ側認証、MVP は匿名)
- **AI Agent Service**:
  - HTTP endpoint: `INGRESS_TRAFFIC_INTERNAL_ONLY` + `allow_unauthenticated = false`
  - Core API から呼び出しは IAM 経由(`roles/run.invoker` を `atm-core-api@` に付与)
  - Pub/Sub pull で起動する場合は IAM 不要(Subscription に push auth を設定するパターンでは必要)

### P-SE-4: Secret Manager の権限制御(UA-N-S-02, UA-N-S-04)
**パターン**:
- 各 secret に `secretAccessor` は **必要な SA だけ**に付与(secret 単位の IAM)
- Secret Manager 監査ログを有効化(`--log-for-actions`)
- ローテーション: Type A(DB パスワード等)は 90 日、Type B(外部 API キー)は手動通知ベース

### P-SE-5: Least Privilege SA(UA-N-S-03)
- ランタイム SA と CI デプロイ SA を分離(tech-stack-decisions.md Section 2.3.7)
- `atm-deploy-core-api` は **対象 Cloud Run のみ**に `run.developer`、他サービスには触れない

### P-SE-6: 監査ログ保持(UA-N-S-04, NDC-2)
- Cloud Logging デフォルト 30 日
- Secret Manager / IAM 変更 / Cloud SQL admin 操作は **Cloud Audit Logs** で自動記録(無料)
- Phase2: `_Required` / `_Default` sink を BigQuery に転送し 1 年保持(Audit のみ)

---

## 5. 監視・通知パターン

### P-MN-1: 構造化ログと集約(UA-N-M-03, NDC-2)
- アプリからは **JSON 形式** でログ出力(stdout/stderr)
- Cloud Run が自動で Cloud Logging に送信
- 保持期間: デフォルト 30 日(NDC-2=A)

### P-MN-2: メトリクス・ダッシュボード(UA-N-M-01)
Cloud Monitoring ダッシュボード 3 枚:
1. **Frontend Overview**: Vercel メトリクス(参照)+ Cloud Logging からの 4xx/5xx 比率
2. **Backend Overview**: Cloud Run(Core API + AI Agent)の CPU / Memory / Request / p95 Latency / Error rate
3. **Database**: Cloud SQL CPU / Memory / Disk / Active connections / Query latency

### P-MN-3: アラートポリシー(UA-N-M-02, NDC-1)
**通知チャンネル**: Email notification channel 1 個。**宛先**: `soneda.yuya@gmail.com`(Terraform 変数 `var.alert_notification_email` で受け取り、`envs/{dev,prod}/terraform.tfvars` で指定)。

```hcl
# modules/shared/observability.tf(抜粋)
resource "google_monitoring_notification_channel" "email" {
  display_name = "Primary Email"
  type         = "email"
  labels = {
    email_address = var.alert_notification_email  # ★ tfvars 経由
  }
}

# アラート例(Cloud Run エラー率)
resource "google_monitoring_alert_policy" "api_error_rate_high" {
  display_name = "atm-api-error-rate-high"
  combiner     = "OR"
  conditions {
    display_name = "5xx rate > 5%"
    condition_threshold {
      filter     = <<EOF
        resource.type = "cloud_run_revision"
        metric.type = "run.googleapis.com/request_count"
        metric.label.response_code_class = "5xx"
      EOF
      comparison = "COMPARISON_GT"
      threshold_value = 0.05
      duration        = "300s"
      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_RATE"
      }
    }
  }
  notification_channels = [google_monitoring_notification_channel.email.id]
}
```

アラート一覧(UA-N-M-02 と同じ):

| アラート | 条件 | 配置 |
|---|---|---|
| `atm-api-error-rate-high` | Cloud Run 5xx 比率 > 5% (5 分) | service module (core-api/monitoring.tf) |
| `atm-api-latency-high` | Cloud Run p95 > 1s (5 分) | service module |
| `atm-sql-connection-exhaustion` | DB 接続数 > max_connections の 80% | shared (database に最も近い) |
| `atm-budget-warning` | 月次コスト $30 到達 | shared (observability.tf) |
| `atm-budget-critical` | 月次コスト $50 到達 | shared |

### P-MN-4: コスト監視(UA-N-C-01, C-02)
- GCP Billing budget アラート($30 警告 / $50 超過)
- Terraform: `google_billing_budget` リソースで宣言
- Vercel 側のコストは外部(Vercel ダッシュボード)で確認

---

## 6. 信頼性・エラー処理パターン

### P-RE-1: アプリ側のリトライ + サーキットブレーカ(AD7=A)
Terraform ではなくアプリコードで実装(U-C, U-D, U-E):
- HTTP クライアントに指数バックオフ(initial 100ms, max 30s)
- 連続失敗で circuit open、半開状態の試行

### P-RE-2: Pub/Sub DLQ(UA-N-P-03)
Topic ごとに **dead letter topic** を設定、subscription 側で最大試行回数を指定。
**Terraform**:
```hcl
# modules/shared/messaging.tf
resource "google_pubsub_topic" "ai_jobs" {
  name = "ai-jobs"
}
resource "google_pubsub_topic" "ai_jobs_dlq" {
  name = "ai-jobs-dlq"
}
```
```hcl
# modules/ai-agent/messaging.tf
resource "google_pubsub_subscription" "ai_jobs" {
  name  = "ai-jobs-worker"
  topic = var.shared_topics.ai_jobs
  dead_letter_policy {
    dead_letter_topic     = var.shared_topics.ai_jobs_dlq
    max_delivery_attempts = 5
  }
  ack_deadline_seconds = 600  # AI 処理は最大 10 分程度/リトライ
}
```

### P-RE-3: Cloud SQL 接続プールリカバリ
- アプリ側でコネクション切断検知 → 自動再接続(pgx, asyncpg 標準機能)
- Cloud SQL メンテナンス通知を受信するよう `maintenance_window` を週次の低利用時間帯に固定

---

## 7. NFR → パターン → Terraform 設定のマッピング

| NFR ID | 対応パターン | Terraform リソース/フラグ |
|---|---|---|
| UA-N-P-01 | P-PF-1 | `startup_cpu_boost = true` |
| UA-N-P-02 | P-SC-3 | `database_flags = [{name="max_connections", value="50"}]` |
| UA-N-P-03 | P-SC-1, P-SC-2, P-RE-2 | `scaling { max_instance_count = 10 }`, Pub/Sub DLQ |
| UA-N-A-01 | P-AV-1 | Cloud Run マネージド配置 |
| UA-N-A-02 | (GCP SLA) | `google_sql_database_instance` Regional 昇格の Phase2 |
| UA-N-A-03/04/05 | P-AV-3 | `backup_configuration { point_in_time_recovery_enabled = true }` |
| UA-N-S-01 | P-SE-1 | `ip_configuration { ipv4_enabled = false; private_network = ... }` |
| UA-N-S-02 | P-SE-4 | `google_secret_manager_secret_iam_member` ごとに最小 SA |
| UA-N-S-03 | P-SE-5 | service module 内の SA + 限定ロール |
| UA-N-S-04 | P-SE-6 | Cloud Audit Logs(デフォルト)+ log sink(Phase2) |
| UA-N-S-05 | P-SE-1 | `ipv4_enabled = false` |
| UA-N-R-01 | P-AV-4 | `traffic` blocks、failure policy |
| UA-N-M-01 | P-MN-2 | `google_monitoring_dashboard` x3 |
| UA-N-M-02 | P-MN-3 | `google_monitoring_alert_policy` + `google_monitoring_notification_channel` |
| UA-N-M-03 | P-MN-1 | Cloud Run デフォルトで自動。アプリ側で JSON 出力 |
| UA-N-C-01/02 | P-MN-4 | `google_billing_budget` |

---

## 8. 未解決事項(Infrastructure Design で確定)

| # | 事項 | 仮置き |
|---|---|---|
| P-OP1 | アラート通知先メール具体値 | `soneda.yuya@gmail.com`(`var.alert_notification_email` 経由、`terraform.tfvars` で指定) |
| P-OP2 | VPC CIDR 具体値 | `10.10.0.0/16` 仮 |
| P-OP3 | dev と prod の Cloud SQL instance 名 | `atm-main-dev`, `atm-main-prod` 仮 |
| P-OP4 | Cloud Run 各サービスの CPU/Memory 最終値 | Section 3 P-PF-3 の値 |
| P-OP5 | Pub/Sub subscription の ack_deadline 最終値 | 600s 仮 |
