# 作業単位 (Unit of Work)

**プロジェクト**: AI Task Manager
**作成日**: 2026-04-19
**分解方針**: 8 ユニット / モノレポ / AI エージェント自動実装 + 並行最大化 / 基盤先行型実装

---

## 1. ユニット一覧サマリ

| ID | ユニット名 | 物理サービス | 主要コンポーネント | 着手順 |
|---|---|---|---|---|
| **U-A** | Infrastructure | Terraform / GCP / Vercel | (横断) | 1 |
| **U-B** | Proto Definitions | Buf workspace | (横断) | 2 |
| **U-H** | CI/CD + Security + PBT Infra | GitHub Actions | (横断) | 3 |
| **U-C** | Core API: Task + Idea + Auth + Audit | Core API (Go) | C-1, C-2(BE), C-7, C-8 | 4 |
| **U-E** | AI Agent Service | AI Agent Service (Python) | C-5, C-6, C-2(AI), C-4(AI) | 5 |
| **U-D** | Core API: Repo + IssuePR + Job + SSE | Core API (Go) | C-3, C-4, C-9, C-10 | 6 |
| **U-G** | GitHub App | App manifest + Webhook | (C-3, C-4 の入口) | 7 |
| **U-F** | Web Frontend | Next.js (Vercel) | 全画面 P-1〜P-12 | 8 |

---

## 2. コード組織戦略(Greenfield モノレポ)

### 2.1 ルートディレクトリ

```
ai-task-manager/
├── proto/                          # U-B: Buf Connect 定義
│   ├── buf.yaml
│   ├── buf.gen.yaml
│   ├── gen/                        # 生成コード(.gitignore)
│   └── <service>/v1/*.proto
├── frontend/                       # U-F: Next.js Web FE
│   ├── src/
│   ├── package.json
│   └── next.config.mjs
├── core-api/                       # U-C, U-D: Go BE
│   ├── cmd/server/main.go
│   ├── internal/
│   ├── migrations/                 # sqlc + migrations
│   └── go.mod
├── ai-agent/                       # U-E: Python AI Agent Service
│   ├── src/
│   ├── pyproject.toml
│   └── uv.lock
├── github-app/                     # U-G: Manifest + Webhook 設定
│   └── manifest.yml
├── infra/                          # U-A: Terraform
│   ├── modules/
│   └── envs/{dev,prod}/
├── .github/workflows/              # U-H: CI/CD
│   ├── frontend.yml
│   ├── core-api.yml
│   ├── ai-agent.yml
│   ├── infra.yml
│   └── proto.yml
├── docs/                           # README、ADR、運用メモ(任意)
├── .gitignore
├── CLOUDE.md                       # AI-DLC ワークフロー定義
├── aidlc-docs/                     # AI-DLC 成果物(本ドキュメント群)
└── README.md
```

### 2.2 モノレポツールの採用

- **ビルドツール**: 採用せず(各言語のネイティブ、各 CI ジョブで対応)
- **型共有**: Buf による `.proto` → 各言語生成で統一
- **依存管理**: 各言語のパッケージマネージャ(npm, go mod, uv)を独立運用

---

## 3. 各ユニット詳細

### U-A: Infrastructure

**目的**: GCP 上のインフラリソースと Vercel プロジェクトを Terraform で宣言的に管理する。
**リポジトリ配置**: `/infra/`

**主要責務**:
- Terraform モジュール化(networking / cloud-sql / cloud-run / pubsub / secret-manager / cloud-storage / iam / monitoring)
- 環境分離(`/infra/envs/dev/`, `/infra/envs/prod/`)
- State backend(GCS バケット)
- Secret Manager シークレットの宣言(実値は手動投入)
- Cloud Monitoring ダッシュボード・アラート定義
- Vercel プロジェクト作成(FE の環境変数、カスタムドメイン)
- VPC / VPC Connector(Cloud Run ↔ Cloud SQL)

**提供物(他ユニットへのインターフェース)**:
- Terraform outputs(Cloud Run URL、Cloud SQL 接続情報、Pub/Sub トピック名、Secret Manager ID など)
- IAM ロール / サービスアカウント(他ユニットがデプロイ時に使用)

**技術スタック**: Terraform 1.9+ / Google provider / Vercel provider
**関連 NFR**: NFR-P-05(水平スケール)、NFR-S-02(Secret Manager)、NFR-M-03(監視)

---

### U-B: Proto Definitions

**目的**: FE ↔ BE、Core API ↔ AI Agent 間の API 契約を Protobuf で宣言し、全言語の型を生成する。
**リポジトリ配置**: `/proto/`

**主要責務**:
- Buf workspace セットアップ(`buf.yaml`, `buf.gen.yaml`, `buf.lock`)
- サービス定義:
  - `task/v1/task.proto`(C-1 関連)
  - `idea/v1/idea.proto`(C-2)
  - `repo/v1/repo.proto`(C-3)
  - `issuepr/v1/issuepr.proto`(C-4)
  - `agent/v1/agent.proto`(C-5, C-10 の Core → AI 同期 RPC)
  - `audit/v1/audit.proto`(C-8)
- 共通型(`common/v1/types.proto`: IDs、ページングカーソル、タイムスタンプ、Error Details)
- Code generation プラグイン設定:
  - Go: `protoc-gen-go` + `protoc-gen-connect-go` → `/core-api/internal/gen/`
  - TypeScript: `@bufbuild/protoc-gen-es` + `@connectrpc/protoc-gen-connect-es` → `/frontend/src/gen/`
  - Python: `protobuf` + `connect-python` → `/ai-agent/src/gen/`
- Buf lint / format / breaking change ルール

**提供物**: 生成済みクライアント/サーバスタブ(各言語)
**技術スタック**: Buf CLI / Protobuf 3

---

### U-H: CI/CD + Security + PBT Infrastructure

**目的**: 継続的な品質ゲート(ビルド・テスト・セキュリティスキャン・PBT・デプロイ)を GitHub Actions で実装する。
**リポジトリ配置**: `/.github/workflows/`

**主要責務**:
- ワークフロー定義:
  - `proto.yml` — Buf lint / generate 差分チェック
  - `frontend.yml` — build / unit test / PBT (fast-check) / Vercel deploy(prod/preview)
  - `core-api.yml` — go build / go test / gopter PBT / govulncheck / Cloud Run deploy
  - `ai-agent.yml` — pytest / hypothesis PBT / pip-audit / Cloud Run deploy
  - `infra.yml` — terraform fmt / validate / plan(PR)/ apply(main merge, 手動承認)
  - `audit-log-export.yml` — 週次の依存脆弱性レポート
- ブランチ保護設定(main への直接 push 禁止、Required checks、Required reviews)
- Dependabot / Renovate 設定(`.github/dependabot.yml`)
- CODEOWNERS 設定(将来のチーム体制に備える)
- コミットサイン(DCO or GPG)方針(任意)
- PBT 基盤:
  - Go: `gopter`(go.mod 依存)
  - Python: `hypothesis`(pyproject 依存)
  - TS: `fast-check`(package.json 依存)
- セキュリティスキャン:
  - `govulncheck`(Go)
  - `pip-audit`(Python)
  - `npm audit` + `osv-scanner`(Node)
  - Trivy(コンテナイメージ)

**提供物**: CI ステータス、デプロイパイプライン、PBT 実行環境
**関連 NFR/US**: NFR-S-05、NFR-T-*、US-6.4, US-6.5

---

### U-C: Core API: Task + Idea + Auth + Audit

**目的**: タスク管理・アイデア対話(BE 側)・認証・監査の Go サーバを実装する。
**リポジトリ配置**: `/core-api/internal/{task,idea,auth,audit,...}/`

**主要責務**:
- Go サービス骨組み(`cmd/server/main.go`、DI コンテナ、Connect RPC サーバ)
- データベース:
  - sqlc + PostgreSQL マイグレーション(`/core-api/migrations/`)
  - テーブル: `tasks`, `subtasks`, `tags`, `task_tags`, `idea_sessions`, `idea_qa_exchanges`, `idea_drafts`, `audit_events`
- ドメイン実装(C-1, C-2 の BE 部、C-7, C-8):
  - Task CRUD、カンバン、楽観ロック、ソフトデリート + 復元
  - Idea 対話セッション管理、ドラフト状態遷移、AI Agent とのコールバック受け口
  - AuthProvider インターフェース + AnonymousProvider(MVP)+ FirebaseAuthProvider(スケルトン)
  - AuthMiddleware(Context に UserID を注入)
  - AuditLogger(全書き込み操作に埋め込み、PostgreSQL + Cloud Logging へ二重書き)
  - AuditQueryService(検索・CSV/JSON エクスポート)
- エラー処理: 指数バックオフ + サーキットブレーカ(外部呼び出し箇所は限定的)
- OpenTelemetry trace 初期化(trace-id を Connect メタデータに伝播)

**提供物**: Connect RPC エンドポイント(Task / Idea / Audit サービス)、Auth Middleware、AuditLogger
**関連 US**: US-1.1〜US-1.10(BE 部)、US-1.11/1.12(BE 部)、US-4.*、US-5.1(書き込み側)
**関連 NFR**: NFR-P-02(p95 200ms)、NFR-S-03(監査)、NFR-T-*(PBT 基盤の使用)

---

### U-D: Core API: Repo + IssuePR + Job Dispatcher + SSE

**目的**: GitHub リポ連携・Issue/PR オーケストレーション・非同期ジョブディスパッチ・SSE 通知を実装する。
**リポジトリ配置**: `/core-api/internal/{repo,issuepr,job,notify,github}/`

**主要責務**:
- ドメイン実装(C-3, C-4, C-9, C-10):
  - GitHub App クライアント(installation token の取得・キャッシュ、API 呼び出し)
  - `GithubWebhookRouter`(HMAC 署名検証、冪等性キー管理、event.id での重複排除)
  - `RepoUsecaseService`(インストール管理、リポ一覧、タスク紐付け)
  - `IssuePRUsecaseService`(Issue/PR の起動、双方向同期)
  - `JobDispatcher`(Pub/Sub パブリッシャ、JobRunRepository)
  - `NotificationHub`(内部イベントバス)+ `SSEHandler`(`/events?channel=...`)
  - `RiskMergePolicyService`(C-6 の結果を受けて分岐)
- DB マイグレーション:
  - テーブル: `installations`, `repo_links`, `job_runs`, `risk_assessments`, `sync_events_seen`
- Pub/Sub トピック / サブスクリプションの利用(U-A で provisioning 済み)

**提供物**: `/webhooks/github`, `/events` (SSE), Connect RPC(Repo / IssuePR / Notify サービス)
**関連 US**: US-2.1〜US-2.5, US-3.4〜US-3.7(マージポリシー側), US-3.8(SSE), US-3.9(ジョブ管理)
**関連 NFR**: NFR-P-04(非同期)、NFR-P-05(水平スケール)、NFR-S-03(監査)

---

### U-E: AI Agent Service

**目的**: Claude Agent SDK を用いた AI エージェント実行(対話生成、Issue 生成、PR 実装、リスク評価、PII フィルタ)を実装する。
**リポジトリ配置**: `/ai-agent/src/`

**主要責務**:
- Python サービス骨組み(`src/main.py`、Litestar or FastAPI、Pub/Sub subscriber)
- Claude Agent SDK インテグレーション:
  - `AgentRunner`(エージェントループ)
  - `ToolRegistry`(GitHub API / ファイル操作 / シェル実行 / Claude サブエージェント呼び出し)
- 特化エージェント:
  - `DialogueAgent`(C-2 AI 部分: 質問生成、提案生成)
  - `IssueGenerationAgent`(C-4 AI 部分: Issue 本文生成)
  - `PRImplementationAgent`(C-4 AI 部分: コード分析・実装・PR 作成)
  - `RiskAssessmentAgent`(C-6: リスク判定サブエージェント、deterministic_signals の計算 + LLM 判定)
- `SandboxExecutorService`(Cloud Run Jobs の一時インスタンス起動、ネットワーク制限、リソース制限)
- `PIIFilter`(正規表現 + 軽量 Claude 分類、入力マスク)
- `JobWorker`(Pub/Sub subscriber、冪等性保証、失敗時の DLQ)
- Core API への結果書き戻し(Connect RPC クライアント)

**提供物**: Pub/Sub ジョブコンシューマ、結果コールバック
**関連 US**: US-1.11(AI 対話側)、US-3.1, US-3.2, US-3.3, US-3.10, US-6.6
**関連 NFR**: NFR-P-04(長時間処理)、NFR-S-06(PII)、NFR-S-07(サンドボックス)

---

### U-F: Web Frontend

**目的**: Next.js + shadcn/ui + Feature-sliced で Web UI を実装する。
**リポジトリ配置**: `/frontend/src/`

**主要責務**:
- Next.js 15 App Router セットアップ(TypeScript、Turbopack、next-themes)
- Tailwind CSS + shadcn/ui 初期化、CSS トークン登録(ui-design.md 準拠)
- Buf Connect TypeScript クライアント統合
- Feature-sliced 実装:
  - `features/tasks/`(P-1 カンバン、P-2 詳細 Sheet)
  - `features/idea-dialogue/`(P-3 AIDLC 形式対話、P-4 ドラフト)
  - `features/ai-runs/`(P-5, P-6 タイムライン)
  - `features/repos/`(P-7, P-8)
  - `features/audit/`(P-9)
  - `features/auth/`(P-12、Phase2)
  - `shared/`(UI 共通、hooks、providers)
- 状態管理: TanStack Query(サーバ状態)+ Zustand(クライアント状態、楽観的更新)
- DnD: dnd-kit(カンバン)
- SSE クライアント: `useNotifications(channel)` hook
- コマンドパレット(⌘+K): `Command` コンポーネント
- レスポンシブ実装(sm〜2xl)、モバイル下部タブバー
- ダークモード切替、アクセシビリティ(WCAG AA)
- Vercel 設定(`vercel.json`、環境変数、プレビューデプロイ)

**提供物**: Web アプリ(Vercel デプロイ)
**関連 US**: Epic 1, 2, 3, 4, 5 の FE 側全て
**関連 NFR**: NFR-P-03(初期レンダリング 1.5s)、アクセシビリティ

---

### U-G: GitHub App

**目的**: GitHub App の manifest・権限・インストールフローを定義する(Webhook 実装本体は U-D)。
**リポジトリ配置**: `/github-app/`

**主要責務**:
- App manifest YAML(`manifest.yml`)
  - 名前、説明、アイコン、Callback URL、Webhook URL
  - 権限: `issues: write`, `pull_requests: write`, `contents: write`(PR 作成)、`metadata: read`
  - 購読イベント: `installation`, `installation_repositories`, `issues`, `pull_request`, `check_run`
- プライベートキー生成 → Secret Manager 登録手順(U-A で枠は作成済み)
- OAuth コールバック URL 設定(Phase2 の GitHub OAuth と重複しない別用途)
- インストール導線(FE の「Install GitHub App」ボタン → GitHub へリダイレクト)の要件定義
- 実運用手順書(README or docs/github-app.md)

**提供物**: GitHub App マニフェスト + 運用手順
**関連 US**: US-2.1, US-2.4(Webhook)

---

## 4. 次ステージへの申し送り

### CONSTRUCTION Per-Unit Loop

各ユニットで以下の 5 ステージを実施(execution-plan.md より):

1. **Functional Design** — ビジネスルール詳細、バリデーション、エッジケース
2. **NFR Requirements** — ユニット固有の NFR 要求(性能・セキュリティ・テスト)
3. **NFR Design** — NFR パターン(サーキットブレーカ、PII フィルタなど)の具体化
4. **Infrastructure Design** — GCP リソースマッピング(U-A が提供するリソースへのバインディング)
5. **Code Generation** — Planning + Generation でコード実装

### Build and Test

- 全ユニットの統合ビルド
- 単体 / 統合 / PBT / セキュリティ / E2E テスト手順書の生成
