# AI 推奨 + 最終確認質問 (Recommendation & Final Clarification)

2 回目の回答で **C2(バックエンド構成)** と **C4a(AI エージェント / LLM)** について AI の解説・推奨を求められたため、各選択肢を比較のうえ推奨を提示します。推奨に同意いただければ簡単にご承認ください。異なる選択を希望される場合はその旨を記入してください。

---

## C2: バックエンド構成の比較と推奨

### 前提条件(これまでの回答から確定している項目)

- **スケール**: MVP から大規模(数千ユーザー、数十万タスク)対応 (C1=C)
- **プラットフォーム**: Web アプリ(MVP) (C3=A)
- **認証最終形**: Google OAuth + GitHub OAuth (C5=B,C)
- **GitHub 連携**: GitHub App 経由 (C6=B)
- **AI エージェント**: マルチリポ対応で GitHub Issue/PR/マージを実施 (C4c=A)
- **FE デプロイ先**: Vercel / **BE デプロイ先**: GCP
- **BE 言語**: Go 希望
- **セキュリティ / PBT 強制**: Yes / Yes

### 選択肢の比較

| 項目 | A) BaaS のみ<br>(Supabase) | B) BaaS + Go<br>(Supabase + Cloud Run) | **C) GCP ネイティブ + Go**<br>(Firebase Auth + Cloud SQL + Cloud Run) | D) Firebase フル |
|---|---|---|---|---|
| Go 希望との整合 | ✗ Go 使えない | △ 部分的 | ✅ フル活用 | ✗ Go 使えない |
| 大規模スケール | △ Edge Functions 制限 | ○ | ✅ 水平スケール自在 | ○ Cloud Functions 制限 |
| GitHub App Webhook | △ | ○ | ✅ Cloud Run + Pub/Sub と相性良い | △ |
| 長時間 AI 処理 | ✗ タイムアウト厳しい | ○ | ✅ Cloud Run jobs / Workflows | ✗ |
| OAuth (Google+GitHub) | ○ Supabase Auth | ○ Supabase Auth | ✅ Firebase Auth 両対応 | ✅ Firebase Auth |
| 開発速度(初速) | ✅ 速い | ○ | △ やや遅い(設計が必要) | ○ |
| コスト予測性 | △ 従量+ベンダーロックイン | ○ | ✅ GCP 内で完結 | △ |
| BaaS との二重管理 | N/A | ✗ 2 基盤管理 | ✅ GCP 内で統一 | N/A |

### 🌟 AI の推奨: **Option C(GCP ネイティブ + Go バックエンド)**

**理由**:
1. **Go 希望とスケール要件の両立**: Cloud Run + Cloud SQL / Firestore なら水平スケール・高負荷にも対応しつつ Go で実装できる
2. **AI エージェント統合との相性**: AI エージェントは LLM 呼び出し・GitHub API 操作など**長時間処理**が発生しやすく、Cloud Run(最大 60分実行可) / Cloud Run Jobs / Cloud Workflows が適する
3. **OAuth の統合**: Firebase Auth は Google / GitHub OAuth の両方をネイティブ対応し、MVP 以降の認証追加もスムーズ
4. **二重管理の回避**: BaaS + 独自バックエンドのハイブリッド(B)は **認証の責務分離**、**DB スキーマ管理の重複**、**監視の分散** など運用コストが上がる
5. **Vercel + GCP の分離**: フロントを Vercel、バックを GCP に分ける構成と整合

**データストアのサブ推奨**:
- **Cloud SQL for PostgreSQL**(推奨): リレーショナル構造が合う(タスク・プロジェクト・ユーザー間の関係が多い)、PBT(プロパティベーステスト)との相性も良い
- Firestore は柔軟だが、複雑なクエリ / JOIN / トランザクションが多い本アプリには不向き

### Question C2-final
Option C(GCP ネイティブ + Go + Cloud SQL for PostgreSQL)で進めてよいですか?

A) はい、Option C で進める(推奨)
B) いいえ、Option B(BaaS + Go)にしたい
C) いいえ、Option A(BaaS のみ / Supabase フル活用)にしたい
D) いいえ、Option D(Firebase フル / Go なし)にしたい
X) Other

[Answer]: 

---

## C4a: AI エージェント / LLM の選択肢の違い

### 各選択肢の概要

#### A) **Claude API(Anthropic API 直接呼び出し)**
- **何をするか**: Anthropic の HTTP API に直接リクエスト、モデル(Opus 4.6 / Sonnet 4.6 / Haiku 4.5)を呼ぶ
- **含まれるもの**: テキスト生成、ツール使用(Tool Use)、プロンプトキャッシュ、Extended Thinking
- **自分で作る必要があるもの**: エージェントループ(ツール実行→結果→次の呼び出し…)、会話履歴管理、エラーリトライ
- **向き**: 完全な制御が必要なカスタムエージェント、ChatGPT-like インターフェース
- **難度**: 中〜高(エージェントロジック自作)

#### B) **Claude Code(CLI エージェント)を内部から呼び出す**
- **何をするか**: `claude` CLI をサブプロセスとして起動、ファイル操作・Bash・Git 操作を任せる
- **含まれるもの**: ファイル読み書き、Bash 実行、Git 操作、すでに成熟したエージェントループ
- **制約**: CLI プロセスを起動するため**サーバーからのマルチテナント実行は難しい**(各リクエストごとにサンドボックス環境が必要)、認証情報の管理が複雑
- **向き**: 開発者のローカル環境、または 1 ユーザー 1 コンテナの構成
- **難度**: 実装は楽だが**運用が難しい**

#### C) **GPT(OpenAI API)**
- **何をするか**: OpenAI の API を利用、GPT-4 / GPT-5 系
- **特徴**: Assistants API、Function Calling あり
- **向き**: Claude の代替、OpenAI エコシステムの活用

#### D) **Claude Agent SDK(推奨)**
- **何をするか**: Anthropic 公式の **エージェント構築 SDK**(Python / TypeScript)。エージェントループ、ツール登録、メモリ、圧縮が組み込み済み
- **含まれるもの**: Tool Use の自動ループ、複数ターン会話、コンテキスト圧縮、サブエージェント、**すべてのサーバーサイド実行を前提に設計**
- **向き**: **本アプリのように GitHub Issue → 実装 → PR → マージを自律実行するワークフロー**
- **難度**: 低〜中(SDK に乗れば楽、Go からは HTTP 経由で呼ぶか Python/TS マイクロサービスに分離)

#### E) **複数 AI を切り替え可能にする**
- **何をするか**: A / C を抽象化レイヤーで切り替え可能にする
- **向き**: ベンダーロックインを避けたい、コスト最適化したい
- **難度**: 高(抽象化で機能差分を吸収する必要あり、例: Claude の Extended Thinking は GPT に無い)

### 🌟 AI の推奨: **Option D(Claude Agent SDK)** をメイン + **Option A(Claude API)** を補助

**理由**:
1. **エージェントループの自作回避**: Issue 読取 → コード分析 → 実装 → PR 作成 → CI 待機 → マージ判断、という長いエージェントワークフローは自作すると複雑。Agent SDK の組み込みループで大幅に楽
2. **本用途との親和性**: Agent SDK は「ツール実行・ファイル編集・外部 API 呼び出し」を前提に設計されており、本アプリの「タスク → 自動実装」フローと合致
3. **C4b=D(リスク判定によるマージ自動/レビュー振り分け)の実現**: SDK のサブエージェント機能でリスク評価エージェントを分離配置できる
4. **Go バックエンドとの統合**: **Agent SDK(Python or TS)専用のマイクロサービス**を 1 つ立て、Go の中核 API から gRPC / HTTP で呼び出す構成が現実的

**結果として推奨する実装構成**:
```
[Go 中核 API (Cloud Run)]
    │
    ├── DB・認証・タスク CRUD
    │
    └─→ [AI Agent マイクロサービス (Cloud Run, Python/TS + Claude Agent SDK)]
              │
              ├── Claude API 呼び出し
              ├── GitHub App API 呼び出し
              └── リスク評価サブエージェント
```

### Question C4a-final
AI エージェント実装について、どれで進めますか?

A) Option D(Claude Agent SDK)をメインに、Python または TypeScript のマイクロサービスとして構築(推奨)
B) Option A(Claude API 直接呼び出し)を Go から直接使う(エージェントループは自作)
C) Option B(Claude Code CLI を内部起動)を試す
D) Option C(GPT / OpenAI API)を使う
E) Option E(複数 AI 切り替え可能)にする
X) Other

[A]: 

---

### Question C4a-lang
AI Agent マイクロサービスを構築する言語は?(A/B が選ばれた場合のみ関係)

A) Python(Claude Agent SDK は Python が最も成熟)
B) TypeScript / Node.js
C) AI の推奨に任せる(→ Python を選択)
X) Other

[A]: 

---

**すべてに回答したら「done」とお知らせください。**
これで要件が確定するので、`requirements.md` の作成に進みます。
