# ユニット生成プラン (Unit of Work Plan)

**プロジェクト**: AI Task Manager
**作成日**: 2026-04-19
**前提**: Application Design(6 ファイル)承認済み

## 目的

システムを **並行開発可能な作業単位(Units of Work)** に分解し、各ユニットを CONSTRUCTION フェーズで per-unit 設計・実装する準備を整える。

Application Design の申し送り(8 候補 U-A〜U-H)を叩き台に、分解の最終確定を行う。

---

## Part 1: Planning — 以下 6 問に回答してください

### Section A: ユニット数と分解方針

#### Question UW1: ユニット数・粒度
Application Design では 8 候補ユニット(U-A〜U-H)を挙げました。どれで進めますか?

A) **推奨: 8 ユニット** のまま進める
  - U-A Infrastructure(Terraform + GCP + Vercel)
  - U-B Proto 定義(Buf Connect スキーマ)
  - U-C Core API: Task + Idea + Auth + Audit
  - U-D Core API: Repo + IssuePR + Job Dispatcher + SSE
  - U-E AI Agent Service(全機能)
  - U-F Web Frontend(全機能)
  - U-G GitHub App 設定・Webhook
  - U-H CI/CD + Security + PBT 基盤
B) **少なめ(4〜5 ユニット)**: FE / Core API / AI Agent / Infra+DevOps の大括り
C) **多め(10+ ユニット)**: 各ドメインコンポーネント(C-1〜C-10)単位に近く分解
X) Other(自分の分解案があれば記入)

[A]: 

---

### Section B: リポジトリ構造

#### Question UW2: リポジトリ形態
コードをどう管理しますか?

A) **モノレポ(推奨)**: 1 つの GitHub リポジトリに frontend/, core-api/, ai-agent/, proto/, infra/ を全部収容。Buf Connect の型共有・CI 統合・アトミックな変更がしやすい
B) **ポリレポ**: 各ユニットを別リポジトリで管理(frontend, core-api, ai-agent, infra など)。独立デプロイが明確だが型共有・変更同期が煩雑
C) **ハイブリッド**: コードはモノレポ、Infra(Terraform)だけ別リポ
X) Other

[A]: 

---

### Section C: チーム・並行開発

#### Question UW3: 開発体制
このプロジェクトは誰が実装しますか?

A) **1 人で順次実装**(個人プロジェクト。並行性より順序重視)
B) **AI エージェントが自動実装**(本アプリの機能を自己適用、Claude Code + AI-DLC ワークフロー)
C) **少人数チーム(2〜3 人)で並行実装**
D) A と B のハイブリッド(人間が主導、AI が補助)
X) Other

[B]: 

---

#### Question UW4: 並行開発の優先度
UW3 で A または B を選んだ場合、並行化よりも **依存関係に沿った順次開発** で問題ありません。UW3=C の場合は以下も回答してください(A/B/D なら「—」で OK):

A) 並行最大化(できる限り同時に動かせる単位を最大化するよう分解)
B) 依存最小化(少数の中核ユニットを先に仕上げ、それから派生ユニット)
C) — (UW3=A/B/D のためこの質問はスキップ)
X) Other

[A]: 

---

### Section D: 実装順序

#### Question UW5: CONSTRUCTION での着手順
8 ユニットのうち、どの順序で CONSTRUCTION(per-unit 設計 + コード生成)を実施しますか?

A) **推奨: 基盤先行型**
  1. U-A Infrastructure(Terraform 骨組み)
  2. U-B Proto 定義(API 契約)
  3. U-H CI/CD 基盤
  4. U-C Core API 前半(Task + Idea + Auth + Audit)
  5. U-E AI Agent Service
  6. U-D Core API 後半(Repo + IssuePR + Job + SSE)
  7. U-G GitHub App
  8. U-F Web Frontend(最後に全体統合)
B) **縦割り型(ユーザー価値先行)**: 1 機能を縦に FE+BE まで貫通 → 次機能 → …(スライス手法)
C) **UI 先行型**: モック UI を先に作り、BE を後からつなぐ
X) Other(順序案を記入)

[A]: 

---

### Section E: コード配置

#### Question UW6: ルートディレクトリ構造
UW2=A(モノレポ)の場合、ルート直下のディレクトリ構造は?(component-dependency.md のパッケージレイアウトを再掲)

A) **推奨**(既に設計済み):
  ```
  /proto/            Buf Connect 定義
  /frontend/         Next.js Web FE
  /core-api/         Go BE
  /ai-agent/         Python AI Agent
  /github-app/       App manifest + Webhook 設定
  /infra/            Terraform
  /.github/workflows/ CI/CD
  ```
B) `services/` の下にすべてのサービスを集約(Nx/Turborepo 系の慣習)
C) `apps/` と `packages/` の分離(pnpm workspace 慣習)
X) Other

[A]: 

---

## Part 2: Generation — 承認後に AI が自動実行

- [x] Step UW-G1: 回答を読み込み分解方針を確定
- [x] Step UW-G2: `aidlc-docs/inception/application-design/unit-of-work.md` を生成
  - [x] 各ユニット(U-A〜U-H)の定義・責務・インターフェース
  - [x] 所属コンポーネント(C-1〜C-10)のマッピング
  - [x] 物理サービスへの配置
  - [x] コード組織戦略(Greenfield モノレポのディレクトリ構造)
- [x] Step UW-G3: `aidlc-docs/inception/application-design/unit-of-work-dependency.md` を生成
  - [x] ユニット間依存行列
  - [x] 実装順序のクリティカルパス
  - [x] 並行開発可能な組み合わせ(UW3=B/UW4=A の AI 並行実装戦略)
- [x] Step UW-G4: `aidlc-docs/inception/application-design/unit-of-work-story-map.md` を生成
  - [x] ユーザーストーリー(US-1.1〜US-6.6)を各ユニットに割当
  - [x] 全ストーリーがいずれかのユニットに含まれることを検証
- [x] Step UW-G5: `aidlc-state.md` を更新(各ユニットを CONSTRUCTION 進捗トラッキングに追加)
- [x] Step UW-G6: 完了メッセージを提示し承認を待つ

---

**全 6 問に回答できたら「done」とお知らせください。**
