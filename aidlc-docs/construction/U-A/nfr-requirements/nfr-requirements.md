# U-A Infrastructure — NFR 要件 (NFR Requirements)

**ユニット**: U-A Infrastructure
**作成日**: 2026-04-19
**前提**: `plans/U-A-nfr-requirements-plan.md` 回答承認済(全 10 問 [A] = 推奨採用)

---

## 1. サマリ

U-A Infrastructure の非機能要求を、プロジェクト全体の NFR(requirements.md Section 5)を満たしつつ **MVP に現実的なコストと複雑度** で定義する。

- **環境**: dev + prod の 2 環境
- **規模**: MVP は db-f1-micro + Cloud Run max 10 から開始
- **可用性**: 99.5%
- **セキュリティ**: VPC 分離、Secret Manager、Least Privilege
- **コスト**: $50/月以下を目標

---

## 2. NFR 一覧(ユニット固有)

| ID | カテゴリ | 要件 | SLI(測定指標) | SLO(目標値) | 関連全体 NFR |
|---|---|---|---|---|---|
| **UA-N-P-01** | パフォーマンス | Cloud Run のコールドスタート時間 | p95 コールドスタート時間 | 2 秒以下 | NFR-P-02 |
| **UA-N-P-02** | パフォーマンス | Cloud SQL 接続プール枯渇を起こさない | 接続失敗率 | 0.01% 以下 | NFR-P-02 |
| **UA-N-P-03** | スケーラビリティ | Cloud Run max-instances で突発トラフィックに対応 | スロットル発生率 | 0.1% 以下 | NFR-P-05 |
| **UA-N-A-01** | 可用性 | Cloud Run サービスの月次可用性 | successful_requests / total_requests | 99.5% 以上 | NFR-A-01 |
| **UA-N-A-02** | 可用性 | Cloud SQL 月次可用性 | GCP SLA 準拠 | 99.95% 以上(GCP 保証) | NFR-A-01 |
| **UA-N-A-03** | 復旧性 | Cloud SQL バックアップからの PITR 可能期間 | 復旧可能ウィンドウ | 7 日以上 | NFR-A-02 |
| **UA-N-A-04** | 復旧性 | DB 障害時の復旧時間(RTO) | 復旧完了までの時間 | 1 時間以内(手動) | — |
| **UA-N-A-05** | 復旧性 | DB 障害時のデータ損失(RPO) | PITR で失われる最大期間 | 5 分以内 | — |
| **UA-N-S-01** | セキュリティ | 全内部通信が VPC 内に閉じる | Public IP 経由の DB アクセス数 | 0 | NFR-S-04 |
| **UA-N-S-02** | セキュリティ | シークレットが Secret Manager に集約 | 環境変数ベタ書きのシークレット数 | 0 | NFR-S-02 |
| **UA-N-S-03** | セキュリティ | 各サービスは Least Privilege SA を利用 | roles/editor 以上の広範ロール付与数 | 0 | NFR-S-04 |
| **UA-N-S-04** | セキュリティ | シークレットアクセスの監査ログ | Secret Manager access log | 全アクセス記録(90 日保持) | NFR-S-03 |
| **UA-N-S-05** | セキュリティ | Cloud SQL への公開 IP 割当禁止 | Public IP 設定数 | 0 | NFR-S-04 |
| **UA-N-R-01** | 信頼性 | Cloud Run のデプロイ失敗時に自動ロールバック | 失敗デプロイの自動切戻成功率 | 100%(Cloud Run 機能利用) | — |
| **UA-N-M-01** | 監視 | 主要メトリクスが Cloud Monitoring で可視化 | ダッシュボード設置数 | 3 以上(FE / BE / DB) | NFR-M-03 |
| **UA-N-M-02** | 監視 | 重大アラートがメールで通知 | アラートポリシー数 | 3 以上(エラー率、レイテンシ、DB 接続) | NFR-M-03 |
| **UA-N-M-03** | 運用 | 構造化ログが Cloud Logging に集約 | JSON log の比率 | 100%(アプリ出力) | NFR-M-02 |
| **UA-N-C-01** | コスト | dev + prod の合計月次コスト | GCP 請求額 + Vercel | $50 以下(MVP) | — |
| **UA-N-C-02** | コスト | コスト超過アラート | 予算アラート閾値 | $30(警告)、$50(超過) | — |

---

## 3. 詳細 NFR 定義

### 3.1 パフォーマンス (UA-N-P-*)

#### UA-N-P-01: Cloud Run コールドスタート
- **背景**: min-instances=0(NR-A3=A)のためコールドスタート発生
- **対策方針**: Go/Python バイナリのサイズ最小化、依存軽量化、必要に応じて `startup CPU boost` 有効化
- **Phase2**: min-instances=1 への昇格でコールドスタート撲滅(予算上昇)

#### UA-N-P-02: Cloud SQL 接続プール
- **背景**: Cloud Run 複数インスタンス × 各インスタンスの接続プール数 × アクティブリクエスト 合計が Cloud SQL の max_connections を超えないこと
- **db-f1-micro**: 標準で max_connections ≒ 25
- **対策方針**: 各 Core API インスタンスあたり接続プール上限 4、max-instances=10 → 最大 40 接続の可能性あり
- **緩和**:
  - max-instances を一時的に 6 に抑える、または Cloud SQL で max_connections パラメータを引き上げ
  - **Phase2**: pgbouncer 導入で接続プール外部化

#### UA-N-P-03: スロットル
- **対策方針**: max-instances=10 + concurrency=80 で理論値 800 同時リクエスト対応(MVP 規模には十分)

### 3.2 可用性 (UA-N-A-*)

#### UA-N-A-01: Cloud Run 可用性 99.5%
- **根拠**: MVP は高可用性より機能実装優先。GCP リージョン単一構成で十分到達可能
- **測定**: Cloud Monitoring の uptime_check または `requests / (requests + errors)`
- **Phase2**: 99.9% へ引き上げ、multi-region 構成検討

#### UA-N-A-03〜05: バックアップ・復旧
- **自動バックアップ**: 毎日 1 回(UTC で DB 低負荷時間帯)
- **PITR**: 有効、保持 7 日
- **手動復旧手順書**: `/docs/runbooks/db-restore.md`(Code Generation で生成)
- **訓練**: dev 環境で月次リストア訓練を推奨(運用後)

### 3.3 セキュリティ (UA-N-S-*)

#### UA-N-S-01: VPC 分離
- **構成**: 
  - VPC `atm-vpc`(リージョン us-central1、または asia-northeast1)
  - サブネット `atm-subnet-private`(Cloud SQL)、Serverless VPC Access Connector `atm-connector`(Cloud Run)
  - Cloud SQL は Private IP のみ割当(Public IP = 無効)
- **通信フロー**: Cloud Run → VPC Connector → Cloud SQL Private IP

#### UA-N-S-02: Secret Manager 一元化
- **対象シークレット**:
  - `github-app-private-key`
  - `anthropic-api-key`
  - `db-password-core-api`
  - `db-password-ai-agent`(将来)
  - `firebase-admin-key`(Phase2 認証)
  - `session-signing-key`(HTTPS セッション)
  - `github-webhook-secret`(HMAC 検証)
- **付与方針**: 各シークレットに専用の IAM バインディング、必要な SA だけに `secretAccessor` 権限

#### UA-N-S-03: Least Privilege SA

ランタイム用 SA と デプロイ用 SA を明確に分離(deploy SA はサービスごとに発行、詳細配置は tech-stack-decisions.md Section 2.3.7 参照):

| SA 名 | 種別 | 配置モジュール | 付与ロール(主要) | 用途 |
|---|---|---|---|---|
| `atm-core-api@` | ランタイム | core-api | `roles/cloudsql.client`, `roles/secretmanager.secretAccessor`(限定), `roles/pubsub.publisher` | Core API(Cloud Run)実行 |
| `atm-ai-agent@` | ランタイム | ai-agent | `roles/run.invoker`(Cloud Run Jobs), `roles/pubsub.subscriber`, `roles/storage.objectUser`(中間生成物), `roles/secretmanager.secretAccessor`(限定) | AI Agent(Cloud Run / Jobs)実行 |
| `atm-terraform@` | CI(横断) | shared | Custom Role(最小化された admin 権限) | Terraform plan/apply(infra.yml) |
| `atm-deploy-core-api@` | CI(service 別) | core-api | `roles/run.developer`(対象 Cloud Run のみ), `roles/iam.serviceAccountUser`(`atm-core-api@` のみ) | Core API のデプロイ(core-api.yml) |
| `atm-deploy-ai-agent@` | CI(service 別) | ai-agent | 同上(対象 Cloud Run + Jobs、`atm-ai-agent@` の impersonate) | AI Agent のデプロイ(ai-agent.yml) |
| `atm-monitoring@`(任意) | 閲覧 | shared | `roles/monitoring.viewer` | 監視ダッシュボード閲覧 |

**WIF bindings**: GitHub Actions からの OIDC token は `attribute.workflow` 属性で workflow 名に紐付け、それ以外の workflow からは impersonate 不可とする(詳細は tech-stack-decisions.md Section 2.3.7)。

### 3.4 監視 (UA-N-M-*)

#### UA-N-M-01: Cloud Monitoring ダッシュボード
- **フロントエンドダッシュボード**: リクエスト数、4xx/5xx、p95 レイテンシ(Vercel メトリクスを参照)
- **バックエンドダッシュボード**: Cloud Run 各サービスの CPU / メモリ / リクエスト / エラー / レイテンシ
- **データベースダッシュボード**: Cloud SQL の CPU / メモリ / ディスク / 接続数 / クエリレイテンシ

#### UA-N-M-02: アラートポリシー
| アラート名 | 条件 | 通知先 |
|---|---|---|
| `atm-api-error-rate-high` | Cloud Run 5xx 比率 > 5%(5 分) | メール |
| `atm-api-latency-high` | Cloud Run p95 > 1s(5 分) | メール |
| `atm-sql-connection-exhaustion` | Cloud SQL 接続数 > max_connections の 80% | メール |
| `atm-budget-warning` | 月次コスト $30 到達 | メール |
| `atm-budget-critical` | 月次コスト $50 到達 | メール + 緊急 |

### 3.5 コスト (UA-N-C-*)

#### UA-N-C-01: $50/月目標の内訳概算(MVP 想定)

| リソース | 月額概算(USD) |
|---|---|
| Cloud Run(Core API + AI Agent、低利用) | $0〜$5(無料枠 200万リクエスト) |
| Cloud SQL db-f1-micro | $10〜$15 |
| Cloud SQL ストレージ 10GB | $2 |
| Serverless VPC Access Connector | $8〜$15 |
| Cloud Storage(中間生成物 5GB 想定) | $0.13 |
| Pub/Sub(低利用) | $0〜$2(無料枠) |
| Secret Manager | $1 未満(アクティブ秘密数で算定) |
| Cloud Logging(50GB 無料枠内) | $0 |
| Cloud Monitoring(無料枠内) | $0 |
| Vercel Hobby(個人利用)または Pro | $0〜$20 |
| **合計見込み** | **$25〜$60/月** |

**注**: VPC Connector が MVP 的にはやや重い。コスト最適化のため **Direct VPC Egress**(Cloud Run 2nd gen の機能)への置換検討余地あり。

---

## 4. 全体 NFR とのトレース

| 全体 NFR | U-A での実現 |
|---|---|
| NFR-P-02(API p95 < 200ms) | Cloud Run リージョン配置 + Cloud SQL Private IP の低遅延 |
| NFR-P-05(水平スケール) | Cloud Run max-instances、Pub/Sub の自動スケール |
| NFR-A-01(月 99.5%) | UA-N-A-01 で直接達成 |
| NFR-S-02(Secret Manager) | UA-N-S-02 で実現 |
| NFR-S-03(監査ログ) | UA-N-S-04、UA-N-M-03 で実現 |
| NFR-S-04(認証・権限) | UA-N-S-01、UA-N-S-03(VPC + Least Privilege) |
| NFR-M-02(構造化ログ) | UA-N-M-03 |
| NFR-M-03(監視) | UA-N-M-01、UA-N-M-02 |

---

## 5. 未解決事項 / 仮定

| # | 事項 | 仮置き | 次ステージで確定 |
|---|---|---|---|
| OP1 | GCP リージョン | `asia-northeast1`(東京) | Infrastructure Design |
| OP2 | VPC CIDR | `10.10.0.0/16` | Infrastructure Design |
| OP3 | DB 名、初期ユーザー | `atm_main`, `atm_app` | Code Generation |
| OP4 | dev と prod の Project 分離 | 2 プロジェクト分離(`atm-dev`, `atm-prod`) | Infrastructure Design |
| OP5 | アラート通知先メール | 未定(ユーザー確認が必要) | NFR Design または Infra Design |
