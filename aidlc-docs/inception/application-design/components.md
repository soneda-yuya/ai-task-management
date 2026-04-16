# コンポーネント定義 (Components)

**設計方針**: ドメインベース(AD1=B)。**10 個の論理コンポーネント**を識別し、物理サービス(FE / Core API / AI Agent Service / GitHub App)にマッピング。
**API 契約**: Buf Connect(Protobuf)で型厳密(AD3)/ 非同期ジョブは Pub/Sub(AD4-C)。

---

## 物理サービス ↔ 論理コンポーネントのマッピング

| 論理コンポーネント | Web FE | Core API (Go) | AI Agent (Python) | GitHub App |
|---|---|---|---|---|
| C-1: Task Management | UI | ✅ 主 | — | — |
| C-2: Idea-to-Task | UI | オーケストレーション | ✅ 生成ロジック | — |
| C-3: Repository Integration | UI | ✅ 主 | — | Webhook 受信 |
| C-4: Issue & PR Orchestration | UI | オーケストレーション | ✅ 実装ロジック | Webhook 受信 |
| C-5: AI Agent | — | クライアント | ✅ 主 | — |
| C-6: Risk Assessment | 表示 | クライアント | ✅ 主 | — |
| C-7: Auth | UI | ✅ 主(Middleware) | クライアント | — |
| C-8: Audit & Observability | UI | ✅ 主 | ログ出力 | ログ出力 |
| C-9: Notification (SSE) | 購読 | ✅ 主(発行) | — | — |
| C-10: Async Job Orchestration | — | ✅ 主(ディスパッチ) | ✅ 実行 | — |

---

## C-1: Task Management コンポーネント

**目的**: タスクの作成・編集・削除・状態管理の中核を担う
**配置**: Core API (Go), Web FE (features/tasks, features/kanban)

**責務**:
- タスクエンティティの CRUD
- カンバン状態遷移(To Do / Doing / Done)
- 優先度・タグ・期限・サブタスクの管理
- アサイン管理
- 楽観的ロックと競合解決(Last-Write-Wins)

**インターフェース** (抽象):
- `TaskRepository`: 永続化
- `TaskService`: ユースケース
- `TaskController` (Connect RPC handler): HTTP エンドポイント
- FE 側: `useTasks()` / `useKanban()` hooks

**関連要件/ストーリー**: FR-T-01〜FR-T-09, US-1.1〜US-1.10

---

## C-2: Idea-to-Task コンポーネント

**目的**: ユーザーのアイデア/ゴールから、AI 対話を経て階層化タスクを提案・確定する
**配置**: Core API(対話セッション管理・ドラフト永続化)+ AI Agent Service(対話・提案生成)+ Web FE (features/idea-dialogue)

**責務**:
- アイデア入力の受付
- 対話セッション(質問/回答ループ)の状態管理
- AI による深掘り質問生成(委譲先: C-5 AI Agent)
- 階層化タスク提案の生成(委譲先: C-5)
- ドラフトタスクの編集・承認・破棄
- 対話履歴とタスクのトレース関係の保存

**インターフェース**:
- `IdeaSessionRepository`, `IdeaDraftRepository`
- `IdeaDialogueService`: 対話のライフサイクル
- `IdeaProposalService`: タスク提案への変換
- FE: `useIdeaDialogue()`, `useDraftReview()` hooks

**関連要件/ストーリー**: FR-T-10, US-1.11, US-1.12

---

## C-3: Repository Integration コンポーネント

**目的**: GitHub App のインストール管理、リポジトリ一覧、タスク ↔ リポ紐付けを提供
**配置**: Core API(永続化・ビジネスロジック)+ GitHub App(Webhook 受信)+ Web FE(設定 UI)

**責務**:
- GitHub App インストール/アンインストールイベントの Webhook 処理
- インストール済みリポ一覧の取得・キャッシュ
- タスクとリポの紐付けレコード管理
- マルチリポジトリに跨るタスク一覧の集約

**インターフェース**:
- `GithubAppClient`: GitHub API ラッパー
- `InstallationRepository`, `RepoLinkRepository`
- `RepoIntegrationService`
- `GithubWebhookHandler` (installation, installation_repositories イベント)

**関連要件/ストーリー**: FR-G-01, FR-G-02, FR-G-05, US-2.1, US-2.2, US-2.5

---

## C-4: Issue & PR Orchestration コンポーネント

**目的**: タスク ↔ Issue ↔ PR の生成・双方向同期をオーケストレーションする
**配置**: Core API(オーケストレーション・永続化)+ AI Agent Service(生成実行)+ GitHub App(Webhook)

**責務**:
- タスクから Issue 作成の起動(非同期ジョブ投入)
- Issue から実装 PR 作成の起動(非同期ジョブ投入)
- GitHub 側のイベント(issues, pull_request)を受信しタスク状態と同期
- 双方向同期の衝突解決(タイムスタンプ最新勝ち)

**インターフェース**:
- `IssuePROrchestrator`
- `IssueSyncService`, `PRSyncService`
- `GithubWebhookHandler` (issues, pull_request イベント)

**関連要件/ストーリー**: FR-G-03, FR-G-04, FR-A-01, FR-A-02, US-2.3, US-2.4, US-3.1, US-3.2

---

## C-5: AI Agent コンポーネント

**目的**: Claude Agent SDK を用いた AI エージェント実行の中核(対話、コード分析、実装、PR 作成)
**配置**: AI Agent Service (Python)

**責務**:
- Claude Agent SDK によるエージェントループの実行
- ツール(GitHub API、ファイル操作、シェル実行サンドボックス)の登録と呼び出し
- アイデア対話での質問生成・提案生成(C-2 のバックエンド)
- Issue/PR 生成(C-4 のバックエンド)
- LLM モデルの用途別選択(Opus/Sonnet/Haiku)

**インターフェース**:
- `AgentRunner`(エージェントループ)
- `DialogueAgent`, `IssueGenerationAgent`, `PRImplementationAgent`(特化エージェント)
- `ToolRegistry`(Claude Agent SDK ツール登録)

**関連要件/ストーリー**: FR-A-01, FR-A-02, FR-T-10 の AI 側, US-1.11, US-3.1, US-3.2

---

## C-6: Risk Assessment コンポーネント

**目的**: 生成された PR のリスクレベル(低/中/高)を判定し、自動マージ / レビュアーアサインの判断材料を提供
**配置**: AI Agent Service(サブエージェント)

**責務**:
- PR の差分・変更ファイル・依存関係を分析
- セキュリティ関連ファイル・DB マイグレーション・認証/認可変更を検知
- リスクレベルと根拠サマリを返す
- 境界値(差分行数、ファイル種別)は決定的テストでカバーされる

**インターフェース**:
- `RiskAssessmentAgent`(Claude Agent SDK のサブエージェント)
- `RiskClassifier`(決定的ロジック部分)
- `RiskReport`(返り値型: level + rationale + affected_areas)

**関連要件/ストーリー**: FR-A-03, US-3.3, US-6.6

---

## C-7: Auth コンポーネント

**目的**: 認証方式の抽象化。MVP は AnonymousProvider、Phase2 で FirebaseAuthProvider を有効化
**配置**: Core API (Go Middleware)、Web FE(authClient)

**責務**:
- `AuthProvider` インターフェース定義(`Authenticate(ctx) -> UserID, error`)
- `AnonymousProvider`(MVP)/ `FirebaseAuthProvider`(Phase2)の実装
- Go HTTP/Connect Middleware で Context に UserID を注入
- FE 側の `authClient` 差し替えで UI 分岐

**インターフェース**:
- `AuthProvider`(Go インターフェース)
- `AuthMiddleware`
- FE: `AuthClient`(TypeScript インターフェース)

**関連要件/ストーリー**: FR-U-01〜FR-U-03, US-4.1〜US-4.4

---

## C-8: Audit & Observability コンポーネント

**目的**: AI アクション、GitHub App イベント、セキュリティイベントの監査ログとシステムメトリクスを管理
**配置**: Core API(書き込み・検索)+ Cloud Logging / Cloud Monitoring

**責務**:
- 構造化監査ログの書き込み(JSON、ISO 8601 タイムスタンプ)
- ユーザー/期間/アクション種別でのフィルタ検索
- CSV/JSON エクスポート
- p95 レイテンシ・エラー率・AI 成功率のメトリクス収集
- アラート設定(Cloud Monitoring)

**インターフェース**:
- `AuditLogger`(全コンポーネントから注入)
- `AuditQueryService`
- `MetricsEmitter`

**関連要件/ストーリー**: NFR-S-03, NFR-M-02, NFR-M-03, US-5.1, US-5.2, US-5.3

---

## C-9: Notification (SSE) コンポーネント

**目的**: AI 実行の進捗や状態変更を FE にリアルタイム通知する
**配置**: Core API(SSE エンドポイント)+ Web FE(EventSource クライアント)

**責務**:
- SSE エンドポイント `/events?channel=ai-run:{id}` の提供
- イベントバス(内部 Pub/Sub)からの購読と配信
- FE 側で TanStack Query のキャッシュ更新

**インターフェース**:
- `NotificationHub`(内部イベントバス)
- `SSEHandler`
- FE: `useNotifications(channel)` hook

**関連要件/ストーリー**: NFR-P-04, US-3.8

---

## C-10: Async Job Orchestration コンポーネント

**目的**: 長時間 AI タスクの非同期実行(Pub/Sub ディスパッチ + Cloud Run Jobs 実行)を管理
**配置**: Core API(ディスパッチ・状態管理)+ AI Agent Service(ワーカー)

**責務**:
- ジョブ定義とキューイング(Pub/Sub トピック)
- ワーカー(Cloud Run Jobs)の呼び出しと状態管理(queued / running / succeeded / failed / timeout)
- タイムアウト(30 分)・リトライ(指数バックオフ)・デッドレター
- ジョブ完了時の通知(C-9 へ転送)

**インターフェース**:
- `JobDispatcher`(Go; Pub/Sub パブリッシャ)
- `JobWorker`(Python; Pub/Sub サブスクライバ)
- `JobRunRepository`(ジョブ実行レコード)

**関連要件/ストーリー**: NFR-P-04, NFR-P-05, US-3.9

---

## 共通: Feature-sliced FE 構造(AD2=A)

Web FE は `src/features/{feature-name}/` 単位で以下を配置:

```
src/
├── features/
│   ├── tasks/           # C-1
│   ├── idea-dialogue/   # C-2
│   ├── kanban/          # C-1 のビュー
│   ├── repos/           # C-3
│   ├── ai-runs/         # C-4, C-9
│   ├── reviews/         # C-6(リスク表示)
│   ├── audit/           # C-8
│   └── auth/            # C-7
├── shared/              # UI ライブラリ、共通 hooks、ユーティリティ
├── gen/                 # Buf Connect 生成コード
└── app/                 # Next.js App Router
```

各 feature 配下に `components/`, `hooks/`, `api/`(Connect RPC クライアント呼び出し), `types/` を配置。
