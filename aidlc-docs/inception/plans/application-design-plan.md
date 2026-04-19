# アプリケーション設計プラン (Application Design Plan)

**プロジェクト**: AI Task Manager
**作成日**: 2026-04-17
**前提**: [requirements.md](../requirements/requirements.md), [stories.md](../user-stories/stories.md), [execution-plan.md](./execution-plan.md) 承認済み

## プランの目的

本プランは、**コンポーネントの識別、責務、インターフェース、サービス層、依存関係** を決定します。
詳細なビジネスロジックは後の Functional Design(CONSTRUCTION, per-unit)で扱います。

---

## Part 1: Planning — 質問に回答してください(8 問)

### Section A: コンポーネントの粒度と境界

#### Question AD1: コンポーネント分割の粒度
以下のいずれが希望に近いですか?

A) **粗め(サービスベース)**: Web FE / Core API / AI Agent Service / GitHub App / Infra の 5 つの大きなコンポーネント(推奨)
B) **中程度(ドメインベース)**: Task / Idea-to-Task / GitHub / AI Agent / Risk / Audit など業務ドメイン単位(8〜10 個)
C) **細かめ(クラス/モジュール単位)**: アプリ設計では機能モジュール単位まで分解(15 個以上)
X) Other

[B]: 

---

#### Question AD2: Frontend の構造
Web フロントエンドの設計スタイルは?

A) **Feature-sliced(推奨)**: 機能単位(features/tasks, features/idea-dialogue, features/kanban)で `components`, `hooks`, `api` を分離
B) **レイヤード**: `pages`, `components`, `api`, `hooks` のレイヤーで横串分離
C) **Atomic Design**: atoms/molecules/organisms/templates/pages の UI 階層
X) Other

[A]: 

---

### Section B: API スタイル

#### Question AD3: FE ↔ BE の API スタイル
Next.js FE と Go BE API の通信スタイルは?

A) **REST + OpenAPI 3.1**(推奨。shadcn/TanStack Query と相性よく、型共有は `openapi-typescript` で自動生成)
B) **tRPC 風(フル TypeScript)** — Go BE のため不採用(NG)
C) **GraphQL**(柔軟だが実装オーバーヘッド大)
D) **gRPC-Web** — 型厳密だが Web では少数派
X) Other

[A,Buff Connectを使用して型定義を丁寧にやりたいです]: 

---

#### Question AD4: Core API ↔ AI Agent Service 間の通信
Go BE と Python AI Agent Service の通信は?

A) **REST / HTTP + JSON**(推奨。シンプル、デバッグしやすい)
B) **gRPC**(型厳密、高性能だが両言語でのツール整備が必要)
C) **Pub/Sub ベース**(非同期 AI ジョブ起動は Pub/Sub、同期は REST のハイブリッド)(推奨される組合せ)
X) Other

[C]: 

---

### Section C: 非同期処理・イベント駆動

#### Question AD5: 長時間 AI タスクの処理パターン
AI エージェントの実行(30 分までの長時間処理)はどう設計しますか?

A) **Pub/Sub + Cloud Run Jobs**(推奨): API は即 202 Accepted、Pub/Sub にジョブ投入、Cloud Run Jobs で実行、完了を DB 書き込み + WebSocket/SSE で通知
B) **Cloud Workflows**: ステップフローを宣言的に定義、可観測性高い、コスト高
C) **シンプル同期 + 長時間リクエスト**(非推奨)
X) Other

[A]: 

---

#### Question AD6: 状態通知(FE への通知)
AI 実行の進捗を FE にどう伝える?

A) **ポーリング(TanStack Query の refetchInterval)**(最もシンプル)
B) **Server-Sent Events(SSE)**(サーバー → クライアントの単方向プッシュ、FE 実装簡単)(推奨)
C) **WebSocket**(双方向、Vercel でのサポート制約あり)
D) **Firebase Realtime Database / Firestore のリアルタイム購読**
X) Other

[B]: 

---

### Section D: 設計パターン・共通基盤

#### Question AD7: エラー処理・再試行
外部 API(Claude、GitHub)呼び出しのエラー処理パターンは?

A) **指数バックオフ + サーキットブレーカ**(推奨): クライアントライブラリでリトライ、連続失敗時はブレーカで遮断
B) **リトライのみ(ブレーカなし)**
C) **デッドレターキュー + 手動再実行**(重要ジョブのみ)
D) A + C を組み合わせる(推奨される組合せ)
X) Other

[A]: 

---

#### Question AD8: 認証抽象層の実装方針
MVP 匿名 → Phase2 OAuth を後付けしやすくするための認証の抽象化は?

A) **Interface + Middleware パターン(推奨)**: `AuthProvider` インターフェース、`AnonymousProvider`/`FirebaseAuthProvider` 実装、Go の Middleware で Context にユーザー ID を注入。FE は `authClient` の差し替えで対応
B) **機能フラグ**: 認証の有無をフラグで切替(コードは単一経路)
C) Phase2 時に大きなリファクタリングを受け入れる(抽象化しない)
X) Other

[A]: 

---

## Part 2: Generation — 承認後に AI が自動実行するステップ

- [x] Step AG1: 回答を読み込み方針確定
- [x] Step AG2: `aidlc-docs/inception/application-design/components.md` — コンポーネント定義・責務・インターフェース
- [x] Step AG3: `aidlc-docs/inception/application-design/component-methods.md` — メソッドシグネチャ(業務ルールは Functional Design へ)
- [x] Step AG4: `aidlc-docs/inception/application-design/services.md` — サービス層・オーケストレーション
- [x] Step AG5: `aidlc-docs/inception/application-design/component-dependency.md` — 依存行列・通信パターン・データフロー
- [x] Step AG6: `aidlc-docs/inception/application-design/application-design.md` — 上記を統合したサマリ
- [x] Step AG7: `aidlc-state.md` 更新
- [x] Step AG8: 完了メッセージ提示

---

**全 8 問に回答できたら「done」とお知らせください。**
曖昧さを分析し、必要なら追加質問、問題なければ設計ドキュメントの生成に進みます。
