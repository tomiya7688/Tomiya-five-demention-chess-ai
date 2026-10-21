# AI Context — Kadoka 5D Chess AI

AI支援開発の**最小入口**。全リポジトリや全設計文書の先読みはしない。

## Project

- Name: Kadoka 5D Chess AI
- Phase: **初期設計／ドキュメント整備**（Core／AI／build／tests は未実装）
- Purpose: 時間・世界線を含むチェスCore、差し替え可能な探索／学習AI、Headless研究基盤
- Candidate stack: C++20 Core／Runtime、Python学習・解析tooling（いずれも予定）
- Document language: **日本語を正本**

## Source of Truth

- 現在の目的・受け入れ条件: 現在のGitHub Issue／明示依頼
- ルール候補と未確定項目: [doc/rules.md](doc/rules.md)
- Architecture／依存境界: [doc/architecture.md](doc/architecture.md)
- AIモデルの実装計画: [doc/ai-model-roadmap.md](doc/ai-model-roadmap.md)
- 作業別の読み方・検証: [doc/context-routing.md](doc/context-routing.md)
- 必要時の作業メモ: [doc/context-pack-template.md](doc/context-pack-template.md)
- 実装後の正本: 対象source＋matching tests（存在するようになってから更新）

## 読む順番

1. 現在task（Issueまたは依頼）。既存の短いTask capsuleがあれば確認。
2. [doc/context-routing.md](doc/context-routing.md)の該当route。
3. `search first, read second` で対象source・対応テスト・直近の契約を絞る。
4. 必要がある場合のみ詳しい仕様、外部参考資料、過去の判断へ。
5. Goal / Required / Acceptance・必須制約・確認方法がそろえば探索停止。

## Invariants

- Coreだけがcanonical state／合法性／ターン確定／終局を決定する。
- AIのmove／turnはproposal。違法提案は正本状態を壊さない。
- 時間移動、世界線分岐、active timeline／Present、ターン全体の合法性を通常チェスとして省略しない。
- Runtimeは軽量。Creator／学習／データ変換から逆依存しない。
- Correctness before strength。固定fixture／seed／探索制限を優先する。
- 未確認ルール・未実装機能は明記し、推測で確定扱いしない。

## Context priority

- **P0**: 現在のtask／acceptance／必須不変条件
- **P1**: 対象source／matching tests／再現fixture
- **P2**: 直接依存／変更対象のinterface
- **P3**: 必要な詳細spec／兄弟repoの該当実装
- **P4**: 全履歴／他のIssue／巨大ログ／生成データ（通常読まない）

## 通常除外

`build/`、`dist/`、`generated/`、学習weight、大規模dataset／JSONL、テスト成功ログ全文、無関係なPR・Issue・docs。

## 検証と報告

まずrouteにある最小の正しい証拠を選ぶ。Core／公開仕様／依存境界の変更なら広げる。**今はコード・テストがないため、ビルド成功を主張しない。**

完了報告: `Changed / Contract / Validation (command, scope, result) / Unverified`。結果が`0 tests`の場合は対象を検証していないと記録する。
