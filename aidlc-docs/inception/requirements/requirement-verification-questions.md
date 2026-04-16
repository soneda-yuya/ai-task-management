# 要件確認のための質問 (Requirements Verification Questions)

以下の質問に順番に回答してください。各質問について、該当する選択肢の文字(A、B、C...)を `[Answer]:` タグの後に記入してください。
どの選択肢も当てはまらない場合は、最後の「Other」を選び、詳細を `[Answer]:` の後に自由記述してください。

---

## 初期意図分析の要約(AI の解釈)

- **ユーザーリクエスト**: 「タスク管理アプリを作りたい。aidlc-rule に沿って要件定義・設計を行い、タスクを管理するアプリケーション。」
- **リクエストタイプ**: 新規プロジェクト(New Project / Greenfield)
- **初期スコープ見積り**: 複数コンポーネント(UI + バックエンド + 永続化層 など)
- **初期複雑度見積り**: Standard(標準) — ただし各質問の回答に応じて調整します

---

## Section A: プロジェクトの基本方針

### Question 1
このアプリは誰が使うことを想定していますか?

A) 個人用(自分だけ)
B) 少人数チーム(家族・友人・小規模チームなど、2〜10人程度)
C) 組織・企業(複数チーム、認証付き)
D) 一般公開(誰でもサインアップ可能)
X) Other (please describe after [Answer]: tag below)

[最初はAだけど後にD]: 

---

### Question 2
どのプラットフォーム / 形態を想定していますか?

A) Web アプリケーション(ブラウザで動作)
B) デスクトップアプリ(macOS/Windows)
C) モバイルアプリ(iOS / Android)
D) CLI(コマンドライン)ツール
X) Other (please describe after [Answer]: tag below)

[BだけどAでも可能。Dも作りたい]: 

---

### Question 3
このプロジェクトの主な目的は何ですか?(最も近いものを1つ)

A) 実際に日常で使う実用的なアプリを作る
B) AI-DLC ワークフローの学習・検証のためのサンプル
C) ポートフォリオや技術検証(PoC)目的
X) Other (please describe after [Answer]: tag below)

[C,複数のAIエージェントがタスク管理をGUIから行い、タスクからGitHubのIssueを作成し、PRを作り、自動でマージまで行うアプリケーションを作りたい]: 

---

## Section B: 機能要件(Functional Requirements)

### Question 4
必須で欲しいコア機能を**複数選択**してください(例: `A,B,C` のようにカンマ区切りで記入)。

A) タスクの作成・編集・削除(CRUD)
B) タスクの完了/未完了の切り替え
C) 期限日の設定とリマインダー
D) タスクの優先度設定(高/中/低 など)
E) タスクのカテゴリ/タグ付け
F) サブタスク(ネスト構造)
G) 他ユーザーへの割り当て(アサイン)
X) Other (please describe after [Answer]: tag below)

[A,B,C,D,E,F,G,Githubとの連携、タスクからの自動実装]: 

---

### Question 5
タスクの表示/整理方法として、どれが必要ですか?(複数選択可、カンマ区切り)

A) シンプルなリスト表示
B) カンバンボード(To Do / Doing / Done の列)
C) カレンダー表示
D) 検索・フィルタ機能(テキスト、タグ、期限など)
E) 並べ替え(作成日、期限、優先度)
X) Other (please describe after [Answer]: tag below)

[カンバンボード]: 

---

### Question 6
認証(ログイン)機能は必要ですか?

A) 不要(ローカルのみで動作、認証なし)
B) 簡易的なユーザー識別のみ(メール/ユーザー名)
C) パスワード認証(メール+パスワード)
D) OAuth(Google、GitHub などのソーシャルログイン)
X) Other (please describe after [Answer]: tag below)

[A一旦は不要、後には欲しい]: 

---

## Section C: 非機能要件(Non-Functional Requirements)

### Question 7
データの永続化はどのように行いますか?

A) ローカルファイル(JSON / SQLite など、ブラウザ外)
B) ブラウザのローカルストレージ / IndexedDB
C) サーバー上のデータベース(PostgreSQL / MySQL など)
D) クラウドサービス(Firebase / Supabase など)
X) Other (please describe after [Answer]: tag below)

[D]: 

---

### Question 8
想定する利用規模(同時利用ユーザー数・タスク件数)はどの程度ですか?

A) 小規模(〜10ユーザー、タスク〜数千件)
B) 中規模(〜数百ユーザー、タスク〜数万件)
C) 大規模(数千ユーザー以上、タスク数十万件以上)
X) Other (please describe after [Answer]: tag below)

[C]: 

---

### Question 9
技術スタックの希望はありますか?(フロントエンド)

A) React(TypeScript)
B) Vue.js
C) Next.js(React ベース、SSR)
D) Svelte / SvelteKit
E) 特にこだわりなし — AI の推奨に任せる
X) Other (please describe after [Answer]: tag below)

[E]: 

---

### Question 10
技術スタックの希望はありますか?(バックエンド、必要な場合)

A) Node.js(Express / Fastify / NestJS)
B) Python(FastAPI / Django)
C) Go
D) バックエンド不要(フロントエンドのみで完結)
E) 特にこだわりなし — AI の推奨に任せる
X) Other (please describe after [Answer]: tag below)

[C]: 

---

### Question 11
デプロイ先の希望はありますか?

A) ローカル実行のみ(配布しない)
B) Vercel / Netlify などの静的/サーバーレスホスティング
C) AWS / GCP / Azure などのクラウド
D) Docker コンテナで自己ホスト
E) 未定 — 後で決める
X) Other (please describe after [Answer]: tag below)

[フロントエンドはVercel,バックエンドはGCP]: 

---

## Section D: 拡張機能のオプトイン

### Question 12: Security Extensions
このプロジェクトでセキュリティ拡張ルールを強制しますか?

A) Yes — すべての SECURITY ルールをブロッキング制約として強制する(本番グレードのアプリに推奨)
B) No — すべての SECURITY ルールをスキップする(PoC、プロトタイプ、実験的プロジェクトに適する)
X) Other (please describe after [Answer]: tag below)

[A]: 

---

### Question 13: Property-Based Testing Extension
このプロジェクトでプロパティベーステスト(PBT)ルールを強制しますか?

A) Yes — すべての PBT ルールをブロッキング制約として強制する(ビジネスロジック、データ変換、シリアライゼーション、状態を持つコンポーネントがあるプロジェクトに推奨)
B) Partial — 純粋関数およびシリアライゼーション往復のみ PBT ルールを強制する(アルゴリズム複雑性が限定的なプロジェクトに適する)
C) No — すべての PBT ルールをスキップする(シンプルな CRUD、UI のみのプロジェクト、重要なビジネスロジックを持たない薄い統合層に適する)
X) Other (please describe after [Answer]: tag below)

[A]: 

---

## Section E: その他 / 自由記述

### Question 14
その他、要件として伝えておきたいこと(デザインの好み、参考にしたいアプリ、避けたい技術、制約条件、期限、予算など)があれば自由記述してください。なければ空欄のままで構いません。

[AIエージェントがタスク管理をGUIから行い、タスクからGitHubのIssueを作成し、PRを作り、自動でマージまで行うアプリケーションを作りたい]: 

---

**回答が終わったら、「done」「完了」「終わりました」などと伝えてください。**
内容に矛盾や曖昧さがあれば追加の質問を作成します。問題なければ要件定義ドキュメントの作成に進みます。
