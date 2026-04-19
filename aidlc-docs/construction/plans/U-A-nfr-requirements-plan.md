# U-A NFR 要件プラン (NFR Requirements Plan)

**プロジェクト**: AI Task Manager
**ユニット**: U-A Infrastructure(Terraform + GCP + Vercel)
**作成日**: 2026-04-19
**前提**: Functional Design はスキップ(U-A は純粋なインフラ、ビジネスロジックなし)
**参考**: [unit-of-work.md](../../inception/application-design/unit-of-work.md), [requirements.md](../../inception/requirements/requirements.md) の Section 5 NFR

## 目的

U-A Infrastructure 自体の非機能要求(スケーラビリティ・可用性・セキュリティ・コスト・運用)を明確化し、Terraform 実装の土台となる技術判断を確定する。

---

## Part 1: Planning — 以下 10 問に回答してください

### Section A: 環境・スケーラビリティ

#### Question NR-A1: 環境数
いくつの環境を立てますか?

A) **MVP は dev + prod の 2 環境**(推奨、シンプル)
B) dev + staging + prod の 3 環境
C) ephemeral + dev + prod(PR プレビュー環境を自動生成)
X) Other

[A]: 

---

#### Question NR-A2: Cloud SQL インスタンスサイズ(本番)
想定する本番 DB のサイズは?(後から変更可能)

A) **最小で開始**: `db-f1-micro`(0.6GB メモリ、無料枠近辺、MVP に適)(推奨で最安)
B) 標準: `db-custom-2-7680`(2 vCPU, 7.5GB RAM)
C) 高性能: `db-custom-4-15360`(4 vCPU, 15GB RAM)
D) 未定 — 後で決める(MVP は A で開始)
X) Other

[A]: 

---

#### Question NR-A3: Cloud Run の並行性
Cloud Run サービスの同時実行数(concurrency)とインスタンス上限は?

A) **推奨(MVP)**: concurrency=80、max-instances=10、min-instances=0(コールドスタート許容)
B) 可用性重視: concurrency=80、max-instances=50、min-instances=1(常時 1 台暖機)
C) 大規模前提: concurrency=100、max-instances=100、min-instances=2
X) Other

[A]: 

---

### Section B: 可用性・復旧

#### Question NR-A4: 可用性目標(SLO)
MVP リリース後の可用性目標は?

A) **月 99.5%**(推奨、MVP 妥当、ダウンタイム約 3.6h/月)
B) 月 99.9%(ダウンタイム約 43分/月、Phase2 目標)
C) 月 99.95%+(プロダクション SaaS 水準、コスト要監視)
X) Other

[A]: 

---

#### Question NR-A5: バックアップ・復旧
Cloud SQL のバックアップ戦略は?

A) **推奨**: 自動バックアップ毎日 1 回、point-in-time recovery 有効、保持 7 日
B) 高頻度: 自動バックアップ毎日 + 手動スナップショット週次、PITR、保持 30 日
C) 最小: 自動バックアップのみ、PITR オフ(コスト最小)
X) Other

[A]: 

---

### Section C: セキュリティ

#### Question NR-A6: ネットワーク分離
Cloud Run ↔ Cloud SQL の接続方式は?

A) **推奨**: VPC + Serverless VPC Access Connector + プライベート IP(Cloud SQL)、インターネット経由しない
B) Cloud SQL Auth Proxy サイドカー(シンプルだが公開 IP 経由)
C) 公開 IP + IAM 認証のみ(最小構成、非推奨)
X) Other

[A]: 

---

#### Question NR-A7: シークレット管理
GitHub App private key、Claude API key、DB パスワード等の管理は?

A) **推奨**: Secret Manager 一元管理、Cloud Run はサービスアカウント経由でランタイム取得、バージョニング有効
B) Secret Manager + KMS カスタム暗号化キー(より厳格)
C) 環境変数ベタ書き(非推奨)
X) Other

[A]: 

---

#### Question NR-A8: IAM ポリシー
GCP リソースへの IAM 権限付与方針は?

A) **推奨**: Least Privilege、各サービスごとに専用 SA(Service Account)、必要最小限のロール
B) より厳格: Custom Role を各 SA ごとに定義し、細粒度で許可
C) 開発スピード優先: roles/editor を広めに付与(本番は NG)
X) Other

[A]: 

---

### Section D: 運用・監視・コスト

#### Question NR-A9: 監視・アラート
どこまで監視するか?

A) **推奨(MVP)**: Cloud Monitoring デフォルトメトリクス + 主要アラート(エラー率 > 5%、p95 レイテンシ > 1s、DB 接続数枯渇)、メール通知
B) 手厚い: + Uptime checks、カスタムダッシュボード、Slack 連携
C) 最小: デフォルトメトリクスのみ、アラートなし
X) Other

[A]: 

---

#### Question NR-A10: コスト上限
MVP の月次コスト目標は?

A) **〜$50/月**(無料枠活用、小さく始める)(推奨で MVP 個人利用)
B) $50〜$200/月(本格運用手前)
C) $200〜$500/月(小規模運用)
D) 気にしない(機能優先)
X) Other(具体的な予算を記入)

[A]: 

---

## Part 2: Generation — 承認後に AI が自動実行

- [x] Step NR-G1: 回答を読み込み NFR 方針確定
- [x] Step NR-G2: `aidlc-docs/construction/U-A/nfr-requirements/nfr-requirements.md` を生成
  - [x] スケーラビリティ、パフォーマンス、可用性、セキュリティ、信頼性、保守性の NFR 一覧
  - [x] 各 NFR の測定指標(SLI)と目標値(SLO)
  - [x] プロジェクト全体の NFR-P/NFR-S/NFR-M とのマッピング
- [x] Step NR-G3: `aidlc-docs/construction/U-A/nfr-requirements/tech-stack-decisions.md` を生成
  - [x] GCP サービス選定の確定(Cloud Run, Cloud SQL, Pub/Sub, Secret Manager, etc.)
  - [x] Terraform バージョン、プロバイダ構成、State backend(GCS)
  - [x] Vercel プロジェクト構成
- [x] Step NR-G4: `aidlc-state.md` を更新

---

**全 10 問に回答できたら「done」とお知らせください。**
