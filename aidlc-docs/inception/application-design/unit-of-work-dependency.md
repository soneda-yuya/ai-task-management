# ユニット依存関係 (Unit of Work Dependency)

**プロジェクト**: AI Task Manager
**作成日**: 2026-04-19

---

## 1. 依存行列 (Dependency Matrix)

行 = 呼び出し元/依存元、列 = 依存先。`→` は直接依存(このユニットの実装開始前、または実装中に依存先から最低限の提供物が必要)。

| | U-A | U-B | U-H | U-C | U-D | U-E | U-F | U-G |
|---|---|---|---|---|---|---|---|---|
| **U-A** | — | | | | | | | |
| **U-B** | | — | | | | | | |
| **U-H** | → (Terraform workflow) | → (Buf 生成確認) | — | | | | | |
| **U-C** | → (Cloud Run / Cloud SQL / Secret Manager) | → (Go gen) | → (CI 実行) | — | | | | |
| **U-D** | → (Pub/Sub / Cloud Run) | → (Go gen) | → (CI 実行) | 間接(DB スキーマ命名規則) | — | | | |
| **U-E** | → (Cloud Run / Cloud Run Jobs / Pub/Sub / Secret Manager / Cloud Storage) | → (Python gen) | → (CI 実行) | | 間接(Pub/Sub トピック名) | — | | |
| **U-F** | → (Vercel プロジェクト) | → (TS gen) | → (CI 実行・Vercel デプロイ) | → (Task/Idea/Audit サービス) | → (Repo/IssuePR/Notify サービス) | | — | |
| **U-G** | → (Secret Manager 枠) | | → (App manifest の検証) | | → (Webhook エンドポイント) | | 間接(インストール導線の FE) | — |

**観察**:
- **U-A / U-B / U-H は基盤ユニット**(他すべてから依存される)
- 循環依存なし(DAG)
- U-C ⇔ U-D は同じ Go バイナリ内に実装されるが、モジュール境界で分離可能(別時期に実装着手可)
- U-F は最後に統合するため多数の依存先を持つ

---

## 2. クリティカルパス

```
U-A (Infrastructure)
   │
   ▼
U-B (Proto Definitions)
   │
   ▼
U-H (CI/CD + Security + PBT)
   │
   ├─────────────┬──────────┐
   ▼             ▼          ▼
U-C (Core API   U-E (AI   [early work on U-F shell possible]
  前半: Task+   Agent
  Idea+Auth+     Service)
  Audit)
   │             │
   └──┬──────────┘
      ▼
U-D (Core API 後半: Repo+IssuePR+Job+SSE)
      │
      ▼
U-G (GitHub App manifest)
      │
      ▼
U-F (Web Frontend 全統合)
```

**クリティカルパス所要(見積もり、MVP ベース)**:
- U-A: 3〜5 日
- U-B: 1〜2 日
- U-H: 2〜3 日
- U-C: 5〜8 日
- U-E: 5〜8 日(U-C と並列可)
- U-D: 4〜6 日
- U-G: 1〜2 日
- U-F: 7〜10 日

**直列に実行した場合**: 合計 約 28〜44 日(2〜3 スプリント)
**並列実施した場合**: 約 18〜28 日(クリティカルパス = U-A → U-B → U-H → U-C → U-D → U-F)

---

## 3. AI エージェントによる並行実装戦略(UW3=B, UW4=A)

本プロジェクトでは **AI エージェントが自動実装**(Claude Code + AI-DLC)を前提としているため、並行性を最大化する。

### 段階別並列実行ポイント

| 段階 | 並列実行するユニット | 根拠 |
|---|---|---|
| Phase 0(基盤) | U-A, U-B, U-H を順次(これらは内部で並列化の余地は小) | 基盤が無いと他が動けない |
| Phase 1(コア並列) | **U-C + U-E + U-F(shell のみ)** を並行エージェントで実装 | Proto 契約がある前提で相互にモックが作れる |
| Phase 2(統合並列) | **U-D + U-G** を並行 | U-C のデータベースパターンが固まったら可 |
| Phase 3(FE 統合) | U-F の機能ごとの feature-slice(tasks / idea-dialogue / ai-runs / repos / audit)を並行 | Feature-sliced 設計のため |

### 並列化の具体例

```
[AI-Agent-1] ──→ U-C 実装
[AI-Agent-2] ──→ U-E 実装           (同時並行)
[AI-Agent-3] ──→ U-F features/tasks
[AI-Agent-4] ──→ U-F features/idea-dialogue
```

すべての実装ブランチは最終的に `main` にマージされ、U-H の CI が統合整合性を検証する。

---

## 4. 実装順序(UW5=A 基盤先行型)

確定した CONSTRUCTION 実行順:

| # | ユニット | 目安工数 | 主な先行条件 | 並列可能な後続 |
|---|---|---|---|---|
| 1 | **U-A** Infrastructure | 3〜5 日 | なし | — |
| 2 | **U-B** Proto Definitions | 1〜2 日 | U-A(GCS state backend) | — |
| 3 | **U-H** CI/CD + Security + PBT | 2〜3 日 | U-A, U-B | — |
| 4 | **U-C** Core API 前半 | 5〜8 日 | U-A, U-B, U-H | **U-E 並行可** |
| 5 | **U-E** AI Agent Service | 5〜8 日 | U-A, U-B, U-H | **U-C 並行可** |
| 6 | **U-D** Core API 後半 | 4〜6 日 | U-C(DB パターン), U-B | **U-G 並行可** |
| 7 | **U-G** GitHub App | 1〜2 日 | U-D(Webhook 実装) | — |
| 8 | **U-F** Web Frontend | 7〜10 日 | U-B, U-C, U-D(最低限の API) | Feature-slice ごとに内部並列 |

---

## 5. リスクと緩和策

| # | リスク | 緩和策 |
|---|---|---|
| D1 | U-A(Terraform)の設計ミスで全デプロイがブロック | 小刻みに plan/apply、モジュール化で部分更新、dev 環境で先行検証 |
| D2 | U-B(Proto)の契約変更が全言語に伝播 | Buf の breaking change 検出を CI に組み込み、メジャーバージョン戦略 |
| D3 | U-C と U-D が同じ Go バイナリ内で相互編集するとマージコンフリクト | ディレクトリ境界を明確化(`internal/{task,idea,...}` vs `internal/{repo,issuepr,...}`)、共有部分(`internal/shared/`)は先に凍結 |
| D4 | U-F の feature 並列実装で UI 一貫性が崩れる | ui-design.md の design tokens と shadcn/ui 共通化、`features/*/components/` は共通 primitive に依存 |
| D5 | U-G の GitHub App 権限申請遅延 | U-G は小粒なので最終統合直前に実施、テスト用アカウントで先行検証 |
