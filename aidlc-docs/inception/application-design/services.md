# サービス層とオーケストレーション (Services & Orchestration)

**目的**: 複数コンポーネントをまたぐユースケースのフロー(オーケストレーション)を定義する。
**サービス境界**: Core API(Go)がオーケストレータ、AI Agent Service(Python)が重い処理の実行者、という役割分担。

---

## サービス一覧

| サービス | 配置 | 主責務 |
|---|---|---|
| **TaskUsecaseService** | Core API | C-1 のユースケース:作成/更新/カンバン DnD/完了切替 |
| **IdeaUsecaseService** | Core API | C-2 のユースケース:対話セッション管理・ドラフト→正式化 |
| **RepoUsecaseService** | Core API | C-3 のユースケース:App 連携、リポ ↔ タスク紐付け |
| **IssuePRUsecaseService** | Core API | C-4 のユースケース:Issue/PR の起動・同期 |
| **RiskMergePolicyService** | Core API | C-6 の結果を受けて「自動マージ / CI 待ち / レビュアーアサイン」を決定 |
| **AuditUsecaseService** | Core API | C-8 のユースケース:監査書き込み・検索・エクスポート |
| **AgentExecutionService** | AI Agent Service | C-5 の実行ランナー:Claude Agent SDK ループ + ツール実行 |
| **SandboxExecutorService** | AI Agent Service | サンドボックス(Cloud Run Jobs 一時インスタンス)管理 |
| **GithubWebhookRouter** | GitHub App (Go の Core API 内) | Webhook 受信→イベント種別ルーティング→各オーケストレータへ |

---

## ユースケースフロー(主要 5 シナリオ)

### シナリオ 1: ユーザーが手動でタスクを作成(US-1.1)

```
FE (features/tasks)
  └→ Connect RPC: TaskService.Create
       └→ TaskUsecaseService.Create
            ├→ TaskRepository.Save
            └→ AuditLogger.Log("task.created")
       ← 返却: Task
  ← FE キャッシュ更新(TanStack Query optimistic)
```

同期処理、p95 < 200ms 目標。

---

### シナリオ 2: アイデアから対話でタスク化(US-1.11 → US-1.12)

```
FE (features/idea-dialogue)
  └→ Connect RPC: IdeaService.StartSession(idea)
       └→ IdeaUsecaseService.StartSession
            ├→ SessionRepository.Save
            └→ JobDispatcher.Dispatch({kind: "idea-dialogue-questions", sid})
       ← 返却: { sessionId, status: "generating-questions" }
  ← FE: SSE 購読開始 (/events?channel=idea:{sid})

[非同期] JobWorker (AI Agent Service)
  ├→ AgentExecutionService.run(DialogueAgent.generate_questions)
  ├→ Core API: PATCH /sessions/{sid} { questions: [...] }
  └→ NotificationHub.Publish("idea:{sid}", { type: "questions-ready" })

[FE が SSE で受信] → 質問 UI 表示 → ユーザー回答
  └→ Connect RPC: IdeaService.AnswerQuestion(ans)
       └→ IdeaUsecaseService.AnswerQuestion
            └→ (深掘り継続 or 「提案へ進む」導線を提示)

[提案フェーズ] Connect RPC: IdeaService.RequestProposal
       └→ JobDispatcher.Dispatch({kind: "idea-proposal", sid})
       ← 返却: { jobId }

[非同期] JobWorker
  ├→ DialogueAgent.propose_tasks(session) → TaskProposal
  ├→ Core API: POST /drafts { proposal, traceMap }
  └→ Publish("idea:{sid}", { type: "draft-ready", draftId })

[FE] ドラフトプレビュー → 編集 → 承認
  └→ Connect RPC: IdeaService.Approve(draftId)
       └→ IdeaUsecaseService.Approve
            ├→ 一括 TaskRepository.Save (親 + サブ)
            ├→ AuditLogger.Log("idea.approved")
            └→ DraftRepository.MarkApproved
       ← 返却: [taskIds]
```

---

### シナリオ 3: タスク → Issue → PR → リスク判定 → 自動マージ(US-2.3, US-3.1〜3.6)

```
FE: 「Issue を作成」ボタン
  └→ Connect RPC: IssuePRService.CreateIssueFromTask(tid)
       └→ IssuePRUsecaseService.CreateIssue
            └→ JobDispatcher.Dispatch({kind: "issue-gen", tid})
       ← 返却: { jobId }

[非同期] JobWorker
  ├→ IssueGenerationAgent.generate_issue_body(task, repo)
  ├→ GithubAppClient.CreateIssue(repo, body) → issue
  ├→ Core API: PATCH /tasks/{tid} { issueId: issue.id }
  └→ Publish("ai-run:{jid}", { type: "issue-created", url })

[ユーザーが「実装を開始」ボタンを押す、または自動ポリシー]
  └→ Connect RPC: IssuePRService.CreatePRFromIssue(iid)
       └→ JobDispatcher.Dispatch({kind: "pr-impl", iid})

[非同期] JobWorker
  ├→ SandboxExecutorService で隔離コンテナ起動(NFR-S-07)
  ├→ PRImplementationAgent.implement_and_open_pr(issue, repo)
  │    ├→ GithubAppClient.CloneRepo → 実装 → コミット → プッシュ
  │    └→ GithubAppClient.CreatePR → pr
  ├→ Publish("ai-run:{jid}", { type: "pr-created", url })
  └→ 続けて → JobDispatcher.Dispatch({kind: "risk-assess", prId})

[非同期] JobWorker (Risk)
  ├→ RiskAssessmentAgent.assess(pr) → RiskReport
  ├→ GithubAppClient.CommentOnPR(report.rationale)
  └→ Core API: POST /risk-assessments { prId, report }
       └→ RiskMergePolicyService.Decide(report, taskPolicy)
            ├→ Low   → GithubAppClient.MergePR (US-3.4)
            ├→ Medium → Wait for CI, on success MergePR (US-3.5)
            └→ High  → GithubAppClient.RequestReview(reviewers) (US-3.6)
```

---

### シナリオ 4: GitHub → アプリ同期(Webhook)(US-2.4)

```
GitHub が Webhook 送信(issues / pull_request / installation)
  ↓
Core API: POST /webhooks/github
  └→ GithubWebhookRouter.Route(event)
       ├→ "installation*" → RepoUsecaseService.HandleInstallation
       ├→ "issues" → IssuePRUsecaseService.SyncIssueState (US-2.4)
       └→ "pull_request" → IssuePRUsecaseService.SyncPRState
  ← 200 OK(冪等性: event.id で重複排除)
```

---

### シナリオ 5: 監査ログ参照(US-5.1)

```
FE (features/audit)
  └→ Connect RPC: AuditService.Search(query)
       └→ AuditUsecaseService.Search → AuditRepository.Query
  ← 返却: AuditEvent[] + cursor

[エクスポート] Connect RPC: AuditService.Export
  └→ ストリーミングレスポンスで CSV/JSON を返却
```

---

## オーケストレーションポリシー

1. **同期 API**: タスク CRUD、セッション状態取得、検索などのレイテンシ critical 操作(p95 < 200ms)
2. **非同期ジョブ**: AI を使う処理すべて(対話生成 / 提案 / Issue 生成 / PR 実装 / リスク判定)
3. **通知**: ジョブ完了は必ず C-9 NotificationHub 経由で SSE 配信
4. **トランザクション境界**: Core API の 1 ユースケース = 1 DB トランザクション。AI Agent Service は冪等性を保証する設計(job id で重複排除)
5. **エラー処理**: AD7=A に従い、指数バックオフ + サーキットブレーカ を Claude/GitHub クライアントに実装

---

## 依存性注入 (DI)

- Go: `wire` または手動 DI コンテナで起動時に組み立て
- Python: FastAPI/Litestar の Depends、もしくはシンプルなファクトリパターン
- テスト時には各 Service の依存(Repository, Client, Agent)をモックに差し替え可能な構成を標準化する
