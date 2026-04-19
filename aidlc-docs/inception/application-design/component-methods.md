# コンポーネントメソッド (Component Methods)

**目的**: 各コンポーネントの主要メソッド(インターフェース)を高レベルで定義する。
**注**: ビジネスルールの詳細は後の **Functional Design(CONSTRUCTION, per-unit)** で決定。ここでは **入出力の型シグネチャと高レベルの目的** に留める。

Connect RPC 契約は Protobuf で記述するが、以下は読みやすさのため **疑似 Go / TypeScript 表記** で掲載する。

---

## C-1: Task Management

### Go サービス層
```go
type TaskService interface {
    Create(ctx, req CreateTaskReq) (Task, error)
    Get(ctx, id TaskID) (Task, error)
    List(ctx, filter TaskFilter) ([]Task, Cursor, error)
    Update(ctx, req UpdateTaskReq) (Task, error)
    Delete(ctx, id TaskID) error
    Restore(ctx, id TaskID) (Task, error)                     // 30 日以内の復元
    ToggleComplete(ctx, id TaskID) (Task, error)
    Move(ctx, id TaskID, column Column, position int) error   // カンバン DnD
    AddSubtask(ctx, parentID TaskID, req CreateTaskReq) (Task, error)
    Assign(ctx, id TaskID, assignee UserID) error
}

type Task struct {
    ID, Title, Description string
    Status Column; Priority Priority; Tags []TagID
    DueDate *time.Time; AssigneeID *UserID
    ParentID *TaskID; Children []TaskID
    CreatedAt, UpdatedAt time.Time; Version int  // 楽観ロック
}
```

### FE (TypeScript hooks)
```ts
useTasks(filter?: TaskFilter): UseQueryResult<Task[]>
useTaskMutations(): { create, update, delete, move, toggle, ... }
```

---

## C-2: Idea-to-Task

### Go
```go
type IdeaDialogueService interface {
    StartSession(ctx, req StartIdeaReq) (Session, error)       // 対話開始
    AnswerQuestion(ctx, sid SessionID, ans Answer) (Session, error)  // 次の質問 or 完了へ
    RequestProposal(ctx, sid SessionID) (Draft, error)         // AI に提案生成を依頼
    EditDraft(ctx, did DraftID, patch DraftPatch) (Draft, error)
    Approve(ctx, did DraftID) ([]TaskID, error)                // 一括正式タスク化
    Discard(ctx, did DraftID) error
    GetSession(ctx, sid SessionID) (Session, error)
}

type Session struct {
    ID; Status (active/draft-ready/approved/discarded)
    InitialIdea string; History []QAExchange
    DraftID *DraftID
}
type Draft struct { ID; ParentTasks []TaskProposal; TraceMap []QAToTask }
```

### Python (AI Agent 側)
```python
class DialogueAgent:
    def generate_questions(session: Session) -> list[Question]: ...
    def propose_tasks(session: Session) -> TaskProposal: ...
```

---

## C-3: Repository Integration

### Go
```go
type RepoIntegrationService interface {
    ListInstallations(ctx, ownerID UserID) ([]Installation, error)
    ListRepos(ctx, inst InstallationID) ([]Repo, error)
    LinkTaskToRepo(ctx, tid TaskID, rid RepoID) error
    UnlinkTaskFromRepo(ctx, tid TaskID) error
}

type GithubWebhookHandler interface {
    HandleInstallation(ctx, event InstallationEvent) error
    HandleInstallationRepos(ctx, event InstallationReposEvent) error
}
```

---

## C-4: Issue & PR Orchestration

### Go
```go
type IssuePROrchestrator interface {
    CreateIssueFromTask(ctx, tid TaskID) (JobID, error)   // 非同期ジョブ投入
    CreatePRFromIssue(ctx, iid IssueID) (JobID, error)    // 非同期
    SyncIssueState(ctx, iid IssueID) error                // Webhook 駆動
    SyncPRState(ctx, pid PRID) error                      // Webhook 駆動
}
```

### Python (AI Agent 側)
```python
class IssueGenerationAgent:
    def generate_issue_body(task: Task, repo: Repo) -> IssueBody: ...

class PRImplementationAgent:
    def implement_and_open_pr(issue: Issue, repo: Repo) -> PRResult: ...
```

---

## C-5: AI Agent

### Python
```python
class AgentRunner:
    def run(prompt, tools, model, max_turns) -> RunResult: ...

class ToolRegistry:
    def register(name, handler, schema): ...
    def invoke(name, args) -> Any: ...
```

---

## C-6: Risk Assessment

### Python
```python
class RiskAssessmentAgent:
    def assess(pr: PRDiff) -> RiskReport: ...

@dataclass
class RiskReport:
    level: Literal["low", "medium", "high"]
    rationale: str
    affected_areas: list[str]      # ["auth", "migration", ...]
    deterministic_signals: dict    # diff_lines, sensitive_files, ...
```

### Go (クライアント)
```go
type RiskClient interface { Assess(ctx, pr PRRef) (RiskReport, error) }
```

---

## C-7: Auth

### Go
```go
type AuthProvider interface {
    Authenticate(ctx, req *http.Request) (UserID, Claims, error)
}

type AnonymousProvider struct{}           // MVP
type FirebaseAuthProvider struct{ ... }   // Phase2

func AuthMiddleware(p AuthProvider) Middleware { ... }
```

### TypeScript (FE)
```ts
interface AuthClient {
  getToken(): Promise<string | null>
  getUser(): User | null
  signIn(provider: "google" | "github"): Promise<void>   // Phase2
  signOut(): Promise<void>
}
```

---

## C-8: Audit & Observability

### Go
```go
type AuditLogger interface {
    Log(ctx, event AuditEvent) error
}

type AuditEvent struct {
    Timestamp time.Time; Actor string; Action string
    Resource string; Metadata map[string]any
}

type AuditQueryService interface {
    Search(ctx, q AuditQuery) ([]AuditEvent, Cursor, error)
    Export(ctx, q AuditQuery, format Format) (io.Reader, error)
}

type MetricsEmitter interface {
    Counter(name string, labels map[string]string, delta int)
    Histogram(name string, labels map[string]string, value float64)
}
```

---

## C-9: Notification (SSE)

### Go
```go
type NotificationHub interface {
    Publish(channel string, event Event) error
    Subscribe(channel string) (<-chan Event, CloseFn)
}

type SSEHandler interface {
    ServeHTTP(w http.ResponseWriter, r *http.Request) // ?channel=ai-run:{id}
}
```

### TypeScript
```ts
useNotifications<T>(channel: string): { events: T[]; isConnected: boolean }
```

---

## C-10: Async Job Orchestration

### Go
```go
type JobDispatcher interface {
    Dispatch(ctx, job JobSpec) (JobID, error)       // Pub/Sub publish
    GetStatus(ctx, id JobID) (JobStatus, error)
    Cancel(ctx, id JobID) error
}

type JobSpec struct {
    Kind JobKind      // "issue-gen" | "pr-impl" | "idea-dialogue" | ...
    Payload []byte
    Timeout time.Duration
    MaxRetries int
}

type JobStatus struct {
    State (queued/running/succeeded/failed/timeout/canceled)
    StartedAt, EndedAt *time.Time
    Progress map[string]any  // C-9 への転送用
    Error *ErrorInfo
}
```

### Python (ワーカー)
```python
class JobWorker:
    def register(kind, handler): ...
    def run(): ...   # Pub/Sub pull & dispatch
```
