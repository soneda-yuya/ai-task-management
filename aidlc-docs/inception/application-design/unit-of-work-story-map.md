# ユーザーストーリー ↔ ユニットのマッピング (Story Map)

**プロジェクト**: AI Task Manager
**作成日**: 2026-04-19
**対象ストーリー**: Epic 1〜6 の全 40 ストーリー

ストーリーは**複数ユニット**に跨る場合があります(BE + FE など)。各 US を実現するために必要な全ユニットを列挙しています。

---

## Epic 1: タスク管理 (Task Management)

| Story ID | 内容(要約) | 主担当ユニット | 補助ユニット |
|---|---|---|---|
| US-1.1 | タスクを作成する | **U-C**(Task API)+ **U-F**(tasks, kanban) | U-B(Proto) |
| US-1.2 | タスクを編集・削除する | **U-C** + **U-F** | U-B |
| US-1.3 | タスクの完了/未完了を切り替える | **U-C** + **U-F** | U-B |
| US-1.4 | タスクに期限とリマインダーを設定する | **U-C** + **U-F** | U-B、(Phase2: 通知送信) |
| US-1.5 | タスクに優先度を設定する | **U-C** + **U-F** | U-B |
| US-1.6 | タスクにタグを付ける | **U-C** + **U-F** | U-B |
| US-1.7 | サブタスクを作成する | **U-C** + **U-F** | U-B |
| US-1.8 | タスクをメンバーにアサインする | **U-C** + **U-F** | U-B(Phase2: 複数ユーザー) |
| US-1.9 | カンバンボードでタスクを可視化する | **U-F** | U-C(List API) |
| US-1.10 | ドラッグ&ドロップで状態変更 | **U-C**(Move API)+ **U-F**(dnd-kit) | U-B |
| US-1.11 | アイデアを AI との対話で深掘りする | **U-C**(Session 管理)+ **U-E**(DialogueAgent)+ **U-F**(P-3 AIDLC UI) | U-B、U-D(JobDispatcher) |
| US-1.12 | AI 提案タスクを編集・承認する | **U-C**(Draft 管理)+ **U-F**(P-4 UI) | U-B |

---

## Epic 2: GitHub リポジトリ連携

| Story ID | 内容 | 主担当ユニット | 補助ユニット |
|---|---|---|---|
| US-2.1 | GitHub App をリポにインストール | **U-G**(App manifest)+ **U-D**(Webhook)+ **U-F**(設定 UI) | U-A(Secret Manager) |
| US-2.2 | タスクにリポを紐付ける | **U-D**(RepoLinkRepository)+ **U-F** | U-C(Task 更新) |
| US-2.3 | タスクから Issue を自動作成 | **U-D**(Orchestrator)+ **U-E**(IssueGenerationAgent)+ **U-F** | U-B、U-G(API 呼び出し基盤) |
| US-2.4 | Issue 状態をタスクと双方向同期 | **U-D**(Webhook Router + SyncService)+ **U-G**(Webhook 受信) | U-C |
| US-2.5 | 複数リポジトリを一元管理 | **U-D**(マルチリポ API)+ **U-F**(集約ダッシュボード) | U-B |

---

## Epic 3: AI エージェント自律実行

| Story ID | 内容 | 主担当ユニット | 補助ユニット |
|---|---|---|---|
| US-3.1 | AI が Issue 本文を自動生成 | **U-E**(IssueGenerationAgent)+ **U-D**(Orchestrator) | U-F(プレビュー UI) |
| US-3.2 | AI がコード実装の PR を作成 | **U-E**(PRImplementationAgent + Sandbox) | U-D、U-A(Cloud Run Jobs)、U-G |
| US-3.3 | リスク評価エージェントが PR リスク判定 | **U-E**(RiskAssessmentAgent + 決定的ロジック) | U-D(結果書き戻し) |
| US-3.4 | 低リスク PR が自動マージ | **U-D**(RiskMergePolicyService)+ **U-E**(評価) | U-G(GitHub API) |
| US-3.5 | 中リスク PR が CI 通過後に自動マージ | **U-D**(check_run Webhook 連携) | U-E、U-G |
| US-3.6 | 高リスク PR に人間レビュアーをアサイン | **U-D**(reviewer assign API) | U-E、U-F(通知) |
| US-3.7 | タスクごとにマージポリシー上書き | **U-C**(Task 拡張)+ **U-F**(設定 UI)+ **U-D**(読み取り) | U-B |
| US-3.8 | AI 実行の進捗を GUI で追跡 | **U-D**(SSE + NotificationHub)+ **U-F**(P-5 タイムライン) | U-E(イベント発行) |
| US-3.9 | AI 実行は非同期で UI をブロックしない | **U-D**(JobDispatcher)+ **U-E**(JobWorker) | U-A(Pub/Sub) |
| US-3.10 | AI が PII を検知してフィルタする | **U-E**(PIIFilter) | U-C(監査ログ) |

---

## Epic 4: 認証 / アカウント

| Story ID | 内容 | 主担当ユニット | 補助ユニット |
|---|---|---|---|
| US-4.1 | MVP 匿名シングルユーザー動作 | **U-C**(AnonymousProvider + Middleware) | U-F(default user 表示) |
| US-4.2 | Google OAuth でログイン(Phase2) | **U-C**(FirebaseAuthProvider)+ **U-F**(ログイン UI) | U-A(Firebase Auth 設定) |
| US-4.3 | GitHub OAuth でログイン(Phase2) | **U-C** + **U-F** | U-A |
| US-4.4 | 認証基盤を最初から組み込む | **U-C**(AuthProvider I/F + Middleware) | U-F(authClient 抽象) |

---

## Epic 5: 監視・監査

| Story ID | 内容 | 主担当ユニット | 補助ユニット |
|---|---|---|---|
| US-5.1 | AI 実行結果の監査ログ | **U-C**(AuditLogger)+ **U-D**(書き込み箇所)+ **U-E**(書き込み箇所)+ **U-F**(P-9 表示) | — |
| US-5.2 | GitHub App アクションの監査ログ | **U-D**(Webhook 時に書き込み)+ **U-G**(manifest) | U-C(AuditLogger) |
| US-5.3 | エラー・レイテンシのメトリクス可視化 | **U-A**(Cloud Monitoring ダッシュボード)+ **U-C**/**U-D**/**U-E**(Emitter) | U-F(リンク) |

---

## Epic 6: 非機能要件(Security & PBT Independent)

| Story ID | 内容 | 主担当ユニット | 補助ユニット |
|---|---|---|---|
| US-6.1 | 全 API は HTTPS のみ | **U-A**(TLS 設定)+ **U-H**(強制設定) | 全サービス |
| US-6.2 | GitHub App 秘密鍵は Secret Manager | **U-A**(Secret Manager)+ **U-D**/**U-E**(読み取り) | — |
| US-6.3 | AI 生成コードはサンドボックス実行 | **U-A**(Cloud Run Jobs)+ **U-E**(SandboxExecutor) | — |
| US-6.4 | 依存脆弱性を CI でスキャン | **U-H**(govulncheck, pip-audit, npm audit) | — |
| US-6.5 | ビジネスロジックに PBT 適用 | **U-H**(PBT 基盤)+ **U-C**/**U-D**/**U-E**/**U-F**(各実装) | — |
| US-6.6 | AI リスク判定に決定的テスト | **U-E**(テストスイート) | U-H(CI 実行) |

---

## ユニット別ストーリー担当サマリ

各ユニットが主担当となるストーリー数(補助のみはカウントせず):

| ユニット | 主担当ストーリー数 | 代表例 |
|---|---|---|
| **U-A** | 4 | US-6.1, US-6.2, US-6.3, US-5.3 |
| **U-B** | 0(基盤) | 全ストーリーの契約定義を供給 |
| **U-H** | 3 | US-6.4, US-6.5, US-6.1 |
| **U-C** | 15 | Epic 1, 4, 5(書き込み側) |
| **U-D** | 13 | Epic 2, 3(オーケストレーション) |
| **U-E** | 8 | Epic 3(AI エージェント全般) |
| **U-F** | 14 | 全画面(P-1〜P-12) |
| **U-G** | 2 | US-2.1, US-5.2 |

**全 40 ストーリーが少なくとも 1 つのユニットに割当てられていることを検証済み** ✅

---

## 新規追加ストーリー(昨日 FR-T-10 対応分)の割当確認

US-1.11(アイデア対話)と US-1.12(ドラフト承認)は **U-C + U-E + U-F** に横断割当済み。これらは C-2 Idea-to-Task コンポーネントの実装にあたり、以下で実現される:

- U-C: `IdeaDialogueService`、`SessionRepository`、`DraftRepository`、`JobDispatcher` 呼び出し
- U-E: `DialogueAgent`(質問生成・タスク提案生成)
- U-F: P-3 AIDLC 形式対話 UI、P-4 ドラフトプレビュー・編集 UI
