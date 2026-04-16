# コンポーネント依存関係 (Component Dependencies)

**目的**: コンポーネント間の依存関係、通信パターン、データフローを可視化する。

---

## 依存行列 (Dependency Matrix)

行が「呼び出し元 / 使用者」、列が「呼び出し先 / 被使用者」。`→` は直接依存。

| | C-1 Task | C-2 Idea | C-3 Repo | C-4 Issue/PR | C-5 Agent | C-6 Risk | C-7 Auth | C-8 Audit | C-9 Notify | C-10 Job |
|---|---|---|---|---|---|---|---|---|---|---|
| **C-1 Task** | — | | | | | | → | → | | |
| **C-2 Idea** | → | — | | | (via C-10) | | → | → | → (SSE) | → |
| **C-3 Repo** | | | — | | | | → | → | | |
| **C-4 Issue/PR** | → | | → | — | (via C-10) | (via C-10) | → | → | → | → |
| **C-5 Agent** | | | | | — | → | | → | | |
| **C-6 Risk** | | | | | | — | | → | | |
| **C-7 Auth** | | | | | | | — | → | | |
| **C-8 Audit** | | | | | | | | — | | |
| **C-9 Notify** | | | | | | | | → | — | |
| **C-10 Job** | | | | | → | → | | → | → | — |

**重要な観察**:
- **C-7(Auth)と C-8(Audit)は横断的関心事**(ほぼすべてのコンポーネントから使われる)
- **C-10(Job Orchestration)は AI 関連のコンポーネント(C-2, C-4)の非同期入口**
- **C-5(Agent)と C-6(Risk)は C-10 経由でのみ起動される**(同期直呼び出しはしない)
- **循環依存なし**(DAG)

---

## 通信パターン

| パターン | 使用箇所 | プロトコル |
|---|---|---|
| **FE ↔ Core API(同期)** | タスク CRUD、セッション操作、検索など | **Buf Connect over HTTPS**(Protobuf 契約) |
| **FE ← Core API(通知)** | AI 実行進捗、状態変化 | **Server-Sent Events(SSE)** |
| **Core API ↔ AI Agent Service(非同期)** | AI 実行ジョブ | **Pub/Sub(Protobuf メッセージ)** |
| **AI Agent Service → Core API(コールバック)** | ジョブ結果書き込み、ドラフト作成 | **Buf Connect(内部)** または **HTTPS PATCH** |
| **GitHub → Core API(Webhook)** | installation / issues / pull_request | **HTTPS POST + HMAC 検証** |
| **Core API / AI Agent → GitHub API** | Issue / PR / マージ操作 | **HTTPS(GitHub App JWT 認証)** |
| **AI Agent → Claude API** | LLM 呼び出し | **HTTPS(Anthropic API キー)** |
| **Core API ↔ Cloud SQL** | 永続化 | **Cloud SQL Proxy + TCP/TLS** |

---

## データフロー図(主要 3 フロー)

### フロー A: アイデア → タスク(US-1.11 / US-1.12)

```
[User]
  │ (1) アイデア入力
  ▼
[FE features/idea-dialogue]
  │ (2) Connect RPC: StartSession
  ▼
[Core API: IdeaUsecaseService]
  │ (3) DB: sessions.insert
  │ (4) Pub/Sub publish: idea-dialogue-questions
  ▼
[AI Agent Service: DialogueAgent]
  │ (5) Claude API(質問生成)
  │ (6) HTTP PATCH /sessions/{sid}
  │ (7) SSE Publish: questions-ready
  ▼
[FE via SSE] ─── (8) 質問 UI 表示 → ユーザー回答 ───
  │ (9) Connect RPC: AnswerQuestion(ans)
  ▼
[Core API]
  │ (10) DB: sessions.update(history)
  ▼ (回答毎に繰り返し 4-10)
...
  │ ユーザーが「提案へ進む」
  ▼
[AI Agent Service: DialogueAgent]
  │ (11) Claude API(タスク提案生成)
  │ (12) HTTP POST /drafts
  │ (13) SSE Publish: draft-ready
  ▼
[FE] ドラフトプレビュー → 編集
  │ (14) Connect RPC: Approve
  ▼
[Core API: IdeaUsecaseService]
  │ (15) DB: tasks.insert(many)
  │ (16) Audit.Log
  │ (17) SSE Publish: tasks-created
  ▼
[FE] カンバンに反映
```

---

### フロー B: タスク → Issue → PR → マージ(US-3.1〜3.6)

```
[User] 「Issue を作成」
  ▼
[FE → Core API] CreateIssueFromTask(tid)
  ▼
[Core API] JobDispatcher.Dispatch("issue-gen") → Pub/Sub
  ▼
[AI Agent Service: IssueGenerationAgent]
  ├→ Claude API(Issue 本文生成)
  ├→ GitHub App API: POST /repos/{owner}/{repo}/issues
  └→ Core API PATCH /tasks/{tid} { issueId }
     + SSE "issue-created"

→ [User] 「実装開始」または自動ポリシー
  ▼
[Core API] Dispatch("pr-impl") → Pub/Sub
  ▼
[AI Agent Service: PRImplementationAgent]
  │ (サンドボックス = Cloud Run Jobs の一時インスタンス)
  ├→ Clone repo → 分析 → 実装 → コミット → プッシュ
  ├→ GitHub App API: POST /repos/.../pulls
  └→ SSE "pr-created"

  ▼
[Core API] 自動で次のジョブ Dispatch("risk-assess")
  ▼
[AI Agent Service: RiskAssessmentAgent]
  ├→ Claude API(差分分析)
  ├→ GitHub App API: POST /repos/.../pulls/{n}/comments
  └→ Core API POST /risk-assessments

  ▼
[Core API: RiskMergePolicyService]
  Decide:
    Low    → GitHub API: PUT /repos/.../pulls/{n}/merge
    Medium → (CI 待ち) GitHub Webhook "check_run.completed" を待ち、成功ならマージ
    High   → GitHub API: POST /repos/.../pulls/{n}/requested_reviewers
  ▼ 結果を SSE で FE に通知
```

---

### フロー C: GitHub → アプリ同期(Webhook, US-2.4)

```
[GitHub] Webhook 送信(issue closed / PR merged / etc.)
  ▼
[Core API: /webhooks/github]
  │ (1) HMAC 署名検証
  │ (2) event.id で冪等性チェック(DB)
  ▼
[GithubWebhookRouter]
  │ ルーティング
  ▼
[対応する UsecaseService.Sync*]
  │ DB 更新(tasks のステータス変更)
  │ Audit.Log
  │ SSE Publish(購読中の FE に通知)
  ▼
[200 OK を GitHub に返却]
```

---

## 共通の横断的関心事(Cross-Cutting Concerns)

- **認証(C-7)**: Go の Middleware で全 Connect RPC に適用、Context に UserID を注入
- **監査(C-8)**: 重要アクションは必ずログ → `AuditLogger.Log`(DB + Cloud Logging)
- **通知(C-9)**: 非同期処理の結果は必ず `NotificationHub.Publish` で発火
- **エラー処理(AD7=A)**: Claude/GitHub クライアントに指数バックオフ + サーキットブレーカを実装
- **トレーシング**: OpenTelemetry で FE → Core API → AI Agent まで trace-id を伝播

---

## パッケージ構造(物理レイアウト)

```
ai-task-manager/
├── proto/                              # Buf Connect の .proto 定義
│   ├── task/v1/task.proto
│   ├── idea/v1/idea.proto
│   ├── repo/v1/repo.proto
│   ├── issuepr/v1/issuepr.proto
│   └── audit/v1/audit.proto
├── buf.yaml, buf.gen.yaml              # Buf 設定
│
├── frontend/                           # Web FE (Next.js + TS)
│   ├── src/features/{tasks,idea-dialogue,kanban,repos,ai-runs,reviews,audit,auth}/
│   ├── src/shared/
│   ├── src/gen/                        # Buf 生成 TypeScript クライアント
│   └── package.json
│
├── core-api/                           # Go BE
│   ├── cmd/server/
│   ├── internal/
│   │   ├── task/      (C-1)
│   │   ├── idea/      (C-2)
│   │   ├── repo/      (C-3)
│   │   ├── issuepr/   (C-4)
│   │   ├── risk/      (C-6 クライアント)
│   │   ├── auth/      (C-7)
│   │   ├── audit/     (C-8)
│   │   ├── notify/    (C-9)
│   │   ├── job/       (C-10 dispatcher)
│   │   └── gen/       (Buf 生成 Go コード)
│   └── go.mod
│
├── ai-agent/                           # Python AI Agent Service
│   ├── src/
│   │   ├── agent/          (C-5)
│   │   ├── dialogue/       (C-2 の AI 部分)
│   │   ├── issue_gen/      (C-4 の AI 部分)
│   │   ├── pr_impl/        (C-4 の AI 部分)
│   │   ├── risk/           (C-6)
│   │   ├── sandbox/        (サンドボックス実行)
│   │   ├── worker/         (C-10 worker)
│   │   └── gen/            (Buf 生成 Python コード)
│   └── pyproject.toml
│
├── github-app/                         # GitHub App 設定・manifest
│   └── manifest.yml
│
├── infra/                              # Terraform
│   ├── modules/
│   └── envs/{dev,prod}/
│
└── .github/workflows/                  # CI/CD
```
