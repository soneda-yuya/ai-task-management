# アプリケーション設計 サマリ (Application Design Summary)

**プロジェクト**: AI Task Manager
**作成日**: 2026-04-17
**本書は以下の設計ドキュメントを統合したサマリです**:

- [components.md](./components.md) — コンポーネント定義
- [component-methods.md](./component-methods.md) — インターフェース
- [services.md](./services.md) — サービス層・オーケストレーション
- [component-dependency.md](./component-dependency.md) — 依存関係・データフロー
- [ui-design.md](./ui-design.md) — UI デザイン(ビジュアル言語・画面インベントリ・ワイヤーフレーム)

---

## 1. 設計決定サマリー

| 決定事項 | 採用 | 根拠 |
|---|---|---|
| コンポーネント粒度 | **ドメインベース 10 個**(AD1=B) | 論理責務を明確化しつつ物理サービス数は少なく保つ |
| FE 構造 | **Feature-sliced**(AD2=A) | 機能単位の独立性が高く、並行開発しやすい |
| FE ↔ BE | **Buf Connect(Protobuf over HTTP)**(AD3=A+) | 型厳密、Go/TS の共通契約、Next.js 互換 |
| Core API ↔ AI Agent | **Pub/Sub(非同期)+ Connect RPC(同期)**(AD4=C) | 長時間 AI 処理を切り離しつつ、軽量な同期呼び出しも可能 |
| 長時間処理 | **Pub/Sub + Cloud Run Jobs**(AD5=A) | 最大 30 分実行、スケールイン・アウト容易、コストも現実的 |
| FE 通知 | **Server-Sent Events(SSE)**(AD6=B) | 単方向プッシュで十分、WebSocket より実装が楽、Vercel 制約なし |
| エラー処理 | **指数バックオフ + サーキットブレーカ**(AD7=A) | 外部 API(Claude/GitHub)の一時障害に耐性 |
| 認証抽象 | **Interface + Middleware**(AD8=A) | MVP 匿名 → Phase2 OAuth の切替を最小変更で実現 |

---

## 2. アーキテクチャ俯瞰図

```
┌────────────────────────────────────────────────────────────┐
│                         User (Web Browser)                 │
└────────────────────────┬───────────────────────────────────┘
                         │ HTTPS
                         ▼
┌────────────────────────────────────────────────────────────┐
│  Web Frontend (Next.js 15, TS)   — Vercel                  │
│  [features/tasks, idea-dialogue, kanban, repos, ai-runs,   │
│   reviews, audit, auth]                                    │
└────┬─────────────────────┬────────────────────────────────-┘
     │ Buf Connect (RPC)   │ SSE (通知購読)
     ▼                     ▼
┌────────────────────────────────────────────────────────────┐
│  Core API (Go, Cloud Run)                                  │
│  ┌────────┬─────────┬─────────┬──────────┬─────────┐       │
│  │ C-1    │ C-2     │ C-3     │ C-4      │ C-7 Auth│       │
│  │ Task   │ Idea    │ Repo    │ IssuePR  │ (MW)    │       │
│  ├────────┴─────────┴─────────┴──────────┴─────────┤       │
│  │ C-8 Audit  │ C-9 Notify(SSE)  │ C-10 Dispatcher │       │
│  └──────┬─────┴──────────────────┴─────────┬───────┘       │
└─────────┼────────────────────────────────-─┼──-────────────┘
          │                                  │
          │ Cloud SQL                        │ Pub/Sub
          ▼                                  ▼
   ┌──────────────┐              ┌────────────────────────┐
   │ PostgreSQL   │              │ AI Agent Service       │
   │ (Cloud SQL)  │              │ (Python, Cloud Run)    │
   └──────────────┘              │  C-5 Agent, C-6 Risk   │
                                 │  C-10 Worker           │
                                 │  [Sandbox via          │
                                 │   Cloud Run Jobs]      │
                                 └──────┬──────────┬──────┘
                                        │          │
                                        ▼          ▼
                                 ┌──────────┐ ┌──────────┐
                                 │Claude API│ │GitHub API│
                                 └──────────┘ └──────────┘
                                                    ▲
                                                    │ Webhooks
                                            ┌───────┴──────┐
                                            │  GitHub App  │
                                            └──────────────┘
```

---

## 3. 10 論理コンポーネント要約

1. **C-1 Task Management** — タスク CRUD、カンバン、サブタスク
2. **C-2 Idea-to-Task** — アイデア → AI 対話 → 提案 → 承認
3. **C-3 Repository Integration** — GitHub App 連携、マルチリポ
4. **C-4 Issue & PR Orchestration** — タスク ↔ Issue ↔ PR 同期
5. **C-5 AI Agent** — Claude Agent SDK によるエージェント実行
6. **C-6 Risk Assessment** — PR リスク判定、マージ可否の材料提供
7. **C-7 Auth** — MVP 匿名 / Phase2 OAuth 抽象層
8. **C-8 Audit & Observability** — 監査ログ・メトリクス
9. **C-9 Notification (SSE)** — FE への進捗リアルタイム通知
10. **C-10 Async Job Orchestration** — 長時間 AI ジョブの非同期実行管理

---

## 4. 物理サービス構成(4 + 1)

| サービス | 言語/技術 | デプロイ先 |
|---|---|---|
| Web Frontend | Next.js 15 (TypeScript) | Vercel |
| Core API | Go + Buf Connect + sqlc | Cloud Run (GCP) |
| AI Agent Service | Python + Claude Agent SDK + FastAPI/Litestar | Cloud Run (GCP) |
| GitHub App | (ホスティングなし、Webhook は Core API が受信) | — |
| Infrastructure | Terraform | GCP + Vercel プロジェクト |

---

## 5. 主要ユースケース(5 シナリオ)

[services.md](./services.md) に詳細:

1. タスク手動作成(同期 p95 < 200ms)
2. アイデア対話 → タスク化(非同期 + SSE)
3. タスク → Issue → PR → リスク判定 → マージ(非同期 + Webhook)
4. GitHub Webhook → アプリ同期
5. 監査ログ参照・エクスポート

---

## 6. 次ステージへの申し送り

### Units Generation(次のステージ)で決めること
- このアプリケーション設計を基に、**作業単位(Units)への分解**を行う
- 候補単位(暫定):
  - U-A: Infrastructure (Terraform + GCP セットアップ + Vercel プロジェクト)
  - U-B: Proto 定義 & 共通型 (Buf Connect スキーマ)
  - U-C: Core API Task + Idea + Auth + Audit
  - U-D: Core API Repo + IssuePR + Job Dispatcher + SSE
  - U-E: AI Agent Service (Agent Runner + Dialogue + IssueGen + PR Impl + Risk + Sandbox)
  - U-F: Web Frontend (Feature-sliced 各 feature)
  - U-G: GitHub App 設定・Webhook ハンドラ
  - U-H: CI/CD + Security スキャン + PBT 基盤

### Functional Design(各 Unit で実施)で決めること
- タスクのバリデーションルール(文字数、日付妥当性、優先度の整合)
- リスク判定の具体しきい値(差分行数、ファイル種別、複合条件)
- 自動マージポリシーの境界ケース(CI 長時間失敗、ブランチ保護違反時のフォールバック)
- 対話の継続/終了判定ロジック(「情報が充分」の判断基準)

### NFR Requirements / Design(各 Unit で実施)で決めること
- 具体的なレート制限の数値(API、Claude、GitHub)
- PII フィルタの実装(正規表現 + 分類器の精度基準)
- キャッシュ戦略(Redis を使うかどうか)
- Sandbox コンテナのリソース制限・ネットワーク制限の具体値

---

## 7. 設計品質チェック

| 項目 | 状態 |
|---|---|
| 要件 FR/NFR はすべて少なくとも 1 コンポーネントで責務を持つ | ✅ components.md 冒頭の関連要件で確認 |
| ユーザーストーリーはすべて対応コンポーネントあり | ✅ トレース可能 |
| 循環依存なし | ✅ DAG 確認済み |
| 横断的関心事が分離されている(Auth / Audit / Notify / Job) | ✅ C-7/C-8/C-9/C-10 |
| スケール設計が NFR-P-05 と整合(水平スケール) | ✅ Cloud Run、Pub/Sub、ステートレス Worker |
| セキュリティ拡張(HTTPS、Secret Manager、サンドボックス、監査)対応 | ✅ 設計に反映 |
| PBT 拡張(決定的テストとプロパティテスト)対応可能な粒度 | ✅ C-6 の deterministic_signals 分離など |
