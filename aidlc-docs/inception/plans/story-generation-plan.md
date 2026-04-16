# ユーザーストーリー生成プラン (Story Generation Plan)

**プロジェクト**: AI Task Manager(仮称)
**作成日**: 2026-04-17
**前提**: [requirements.md](../requirements/requirements.md) 承認済み、[user-stories-assessment.md](./user-stories-assessment.md) により User Stories ステージ実施を承認

## プランの目的

本プランは、要件ドキュメントから **ペルソナ** と **ユーザーストーリー** を生成するための手順を定義します。
ユーザーが以下の質問に回答することで、生成方針を確定させ、それに基づいてストーリーを生成します。

---

## Part 1: Planning(現段階)— 以下の質問に回答してください

### Section A: ペルソナ設計

#### Question P1: 想定するペルソナ
生成すべきペルソナを**複数選択**してください(カンマ区切り)。MVP で登場する役割のみで十分です。

A) 個人開発者(Solo Developer)— 自分のリポジトリでタスク→実装を回したい
B) 小規模チームのリーダー — チームのタスクを管理、AI エージェントの出力をレビュー
C) コードレビュアー — 高リスク PR をレビュー・承認する
D) Organization 管理者 — GitHub App のインストール・権限スコープを管理
E) 外部のエンドユーザー(一般ユーザー) — 機能を使うだけ、GitHub は触らない
X) Other

[Answer]: 

---

#### Question P2: ペルソナごとに含める情報
各ペルソナに含める情報のレベルは?

A) 最小(名前、役割、主なゴール)
B) 標準(+モチベーション、フラストレーション、技術的リテラシー)
C) 詳細(+デモグラフィック、経験年数、利用デバイス、好みのワークフロー、KPI)
X) Other

[Answer]: 

---

### Section B: ストーリーの粒度と形式

#### Question P3: ストーリーの粒度
ユーザーストーリーの粒度はどの程度にしますか?

A) **大きめ(エピック寄り)** — 1 ストーリー = 1〜2 スプリント相当、全体像を把握しやすい
B) **標準(推奨)** — 1 ストーリー = 2〜5 日で実装可能、INVEST の "Small" を満たす
C) **細かめ** — 1 ストーリー = 半日〜1 日、タスクボード運用に適する
X) Other

[Answer]: 

---

#### Question P4: ストーリーの書式(テンプレート)
以下のどの書式でストーリーを記述しますか?

A) 標準形式: 「**〜として、〜のために、〜がしたい**」(As a [persona], I want to [action], so that [benefit])
B) Connextra 形式 + 受入条件 Gherkin: As a... I want... so that... + Given/When/Then
C) Job Stories 形式: 「**〜の状況で、〜をしたい、それは〜のためだ**」(When [situation], I want to [motivation], so I can [outcome])
X) Other

[Answer]: 

---

### Section C: 分類 / 構造

#### Question P5: ストーリーの整理方法
ストーリーをどう整理しますか?

A) **ユーザージャーニー順**: ユーザーの時系列に沿って(登録→初期設定→タスク作成→AI 実行→レビュー)
B) **機能ドメイン別**: タスク管理 / GitHub 連携 / AI エージェント / 認証 / 管理画面 のドメイン別
C) **ペルソナ別**: 各ペルソナごとにストーリーをまとめる
D) **ハイブリッド(推奨)**: 機能ドメイン別(B)を主軸に、各ドメイン内でペルソナを明示
X) Other

[Answer]: 

---

#### Question P6: エピック(親ストーリー)を使うか
エピック(親)→ ストーリー(子)の階層構造を採用しますか?

A) はい、エピックを使う(ドメインごとにエピックを立てる)
B) いいえ、フラットなストーリーのみ
X) Other

[Answer]: 

---

### Section D: 受入条件(Acceptance Criteria)

#### Question P7: 受入条件の書式
受入条件の書式は?

A) **Given/When/Then**(Gherkin 形式、BDD スタイル、推奨)
B) **チェックリスト**(受入条件を箇条書き)
C) **自由記述**(ストーリーの中に自然文で埋め込む)
X) Other

[Answer]: 

---

#### Question P8: 受入条件の詳細度
各ストーリーに含める受入条件の項目数の目安は?

A) 最小(1〜2 項目、ハッピーパスのみ)
B) 標準(3〜5 項目、ハッピーパス + 主要な異常系)
C) 詳細(6+ 項目、ハッピーパス + 異常系 + エッジケース + パフォーマンス基準)
X) Other

[Answer]: 

---

### Section E: 特殊ストーリー

#### Question P9: AI エージェントの挙動に関するストーリー
AI エージェントがどう動くべきかを**システム視点のストーリー**として含めますか?(例:「システムとして、生成されたコードの差分サイズから PR リスクを判定したい」)

A) はい、システムストーリーとして含める(AI の自律動作の境界を明確化)
B) いいえ、ユーザー視点のストーリーに暗黙的に含める
X) Other

[Answer]: 

---

#### Question P10: セキュリティ / PBT ストーリー
セキュリティ / プロパティベーステストの要件を**独立したストーリー**として含めますか?

A) はい、独立したストーリーにする(各拡張要件を明示化、監査性が向上)
B) いいえ、受入条件の中に含める(冗長さを避ける)
X) Other

[Answer]: 

---

### Section F: 優先度 / MoSCoW

#### Question P11: ストーリーに優先度を付けるか
各ストーリーに優先度(Must/Should/Could/Won't または MVP/Phase2)を付けますか?

A) はい、MoSCoW(Must/Should/Could/Won't have)で付ける
B) はい、MVP / Phase2 の 2 段階で付ける
C) いいえ、優先度なしでフラットに並べる
X) Other

[Answer]: 

---

### Section G: その他

#### Question P12
その他、ストーリー生成に関して希望があれば自由記述してください(書式テンプレート、参考事例、避けたい表現など)。

[Answer]: 

---

## Part 2: Generation — 承認後に AI が自動実行するステップ

以下は Part 1 の質問に回答・承認後に AI が実行する手順です。ユーザーは回答のみで OK。

- [ ] Step G1: 回答を読み込み、承認されたメソドロジーを確定
- [ ] Step G2: `aidlc-docs/inception/user-stories/personas.md` を生成(P1, P2 に基づく)
- [ ] Step G3: エピック構造を決定(P6)、ドメイン分類を確定(P5)
- [ ] Step G4: `aidlc-docs/inception/user-stories/stories.md` を生成
  - [ ] P4 のテンプレートでストーリーを記述
  - [ ] P7 / P8 の書式・詳細度で受入条件を記述
  - [ ] P9 に従い AI エージェント挙動のシステムストーリーを(選択時は)含める
  - [ ] P10 に従いセキュリティ / PBT 要件を(選択時は)独立ストーリー化
  - [ ] P11 の方法で優先度を付与
- [ ] Step G5: INVEST(Independent / Negotiable / Valuable / Estimable / Small / Testable)チェックを全ストーリーに適用
- [ ] Step G6: 要件ドキュメントの各 FR/NFR と stories の対応マップを stories.md に含める(トレーサビリティ)
- [ ] Step G7: ペルソナ→関連ストーリーのマッピングを personas.md に含める
- [ ] Step G8: `aidlc-docs/aidlc-state.md` の進捗を更新
- [ ] Step G9: 完了メッセージを提示、ユーザー承認を待つ

---

**Part 1 の全質問(P1〜P12)に回答できたら「done」とお知らせください。**
回答の曖昧さを分析して、必要なら追加質問を作成します。問題なければストーリー生成(Part 2)に進みます。
