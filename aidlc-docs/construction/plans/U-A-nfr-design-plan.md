# U-A NFR 設計プラン (NFR Design Plan)

**プロジェクト**: AI Task Manager
**ユニット**: U-A Infrastructure
**作成日**: 2026-04-19
**前提**: [nfr-requirements.md](../../construction/U-A/nfr-requirements/nfr-requirements.md) 承認済

## 目的

NFR 要件を満たすための **パターンと論理コンポーネント** を具体化する。多くは Terraform フラグや GCP 機能の選択に落とし込まれる。

---

## Part 1: Planning — 以下 6 問に回答してください(すべて推奨値あり)

### Section A: 可用性・レジリエンス

#### Question NDA-1: 可用性戦略(HA)
Cloud Run + Cloud SQL の可用性構成は?

A) **推奨(MVP)**: 単一リージョン(`asia-northeast1`)のマルチゾーン自動配置(Cloud Run は自動、Cloud SQL は Regional availability=ZONAL)
B) Regional HA: Cloud SQL を `availability_type=REGIONAL` に(セカンダリゾーンに同期レプリカ、+$10-15/月)
C) Multi-region: 本格 DR 構成(MVP では過剰)
X) Other

[A]: 

---

#### Question NDA-2: コールドスタート緩和
Cloud Run コールドスタート(UA-N-P-01)への対処は?

A) **推奨(MVP)**: `startup CPU boost` 有効、バイナリ軽量化で対応(min-instances=0 を維持)
B) min-instances=1 に昇格して暖機を保証(+$15-30/月)
C) 何もしない(コールドスタートを許容)
X) Other

[A]: 

---

### Section B: セキュリティ設計パターン

#### Question NDB-1: GitHub Actions → GCP 認証
CI/CD(U-H)から GCP へのデプロイ認証は?

A) **推奨**: Workload Identity Federation(長期 SA キー不要、GitHub OIDC 利用、セキュリティ高)
B) SA JSON キーを GitHub Secrets に保存(シンプルだが鍵漏洩リスク)
X) Other

[A]: 

---

#### Question NDB-2: Cloud Run 公開範囲
Core API と AI Agent Service の Cloud Run サービスは?

A) **推奨**: 
  - Core API = パブリック(FE から直接呼ばれるため、`--allow-unauthenticated` + アプリ側認証)
  - AI Agent Service = 内部のみ(`--no-allow-unauthenticated` + IAM 経由の Core API からの呼び出しのみ、または Pub/Sub pull のみ)
B) 両方パブリック(危険、非推奨)
C) 両方 IAP(Identity-Aware Proxy)で保護(MVP の匿名動作と相性悪い)
X) Other

[A]: 

---

### Section C: 運用・通知

#### Question NDC-1: アラート通知先
UA-N-M-02 のアラート(エラー率、レイテンシ、DB 接続、コスト)の通知先メールアドレスは?

A) 使用する Gmail アドレス(自由記述で指定してください)
B) 通知チャンネルは後で追加(Terraform では変数として外出し)
X) Other(Slack や PagerDuty 等)

[A]: soneda.yuya@gmail.com 

---

#### Question NDC-2: ログ保持期間
Cloud Logging の保持期間は?

A) **推奨(MVP)**: デフォルト 30 日(Cloud Logging 標準、無料)
B) 90 日に延長(監査要件対応、軽度追加コスト)
C) 1 年(Audit Log のみ BigQuery へ sink、他は 30 日)
X) Other

[A]: 

---

## Part 2: Generation — 承認後に AI が自動実行

- [x] Step ND-G1: 回答を読み込み方針確定(NDC-1 のメール値は var 外出し)
- [x] Step ND-G2: `aidlc-docs/construction/U-A/nfr-design/nfr-design-patterns.md` を生成
  - [x] レジリエンス、スケーラビリティ、パフォーマンス、セキュリティの設計パターン
  - [x] 各 NFR を満たすための Terraform リソース設定マッピング
- [x] Step ND-G3: `aidlc-docs/construction/U-A/nfr-design/logical-components.md` を生成
  - [x] VPC、VPC Connector、SA、Secret、Pub/Sub 等の論理コンポーネント定義
  - [x] 各コンポーネントの関係・連携パターン
- [x] Step ND-G4: `aidlc-state.md` を更新

---

**全 6 問に回答できたら「done」とお知らせください。**
