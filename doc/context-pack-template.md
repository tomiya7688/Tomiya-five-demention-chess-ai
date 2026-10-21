# Task Context Pack — テンプレート

AI Context Reducer（ObaPick）の [CONTEXT_PACK.md](https://github.com/tomiya7688/ai-context-reducer/blob/main/templates/CONTEXT_PACK.md) を参考にした**一作業用の小さい索引**。テンプレートは原典の代替ではない。

作業が複雑な場合だけこのひな型をコピーし、`.codex/next_issue.md`など**作業環境内**に保存する。必要ならチャット内の短い箇条書きだけでもよい。毎task巨大なファイルを自動生成しない。生成したメモは古くなり得るため、次の作業では現在のIssue、source、testsを優先する。

## Task

- Issue / request:
- Goal:
- Required:
- Acceptance:
- Out of scope:

## Working Set

- Route: `doc/context-routing.md` の該当route
- Target files / symbols:
- Matching tests / fixtures:
- Direct dependencies / external contracts:
- First-read source:
- Read only if needed:

## Constraints and Decision

- 守るinvariant:
- 今回の決定（確定／提案／未確認）:
- 判断根拠（原典のパス・見出し／外部URL）:
- 変更が広がった場合の確認先:

## Evidence Ledger

- Required evidence:
- Known:
- Missing:
- Conflicts:
- Unverified:
- 探索停止条件（何が分かれば実装に進めるか）:

## Validation

- Targeted tests / fixtures:
- 拡張する条件:
- Build / headless / benchmark / artifact validation（該当分だけ）:
- 実行結果（コマンド・対象・件数・結果）:
- 未実施と理由:

## Change Summary

- Changed files:
- Diff stat:
- 変更した契約と影響:
- Validation summary:

## 使い方と判断基準

- Search first, read second。taskと無関係な全repo、履歴、全doc、巨大生成データは入れない。
- Source抜粋は必要最小限とし、重要な判断は原典へ戻る。
- 修正箇所とAcceptance判断に必要な根拠がそろえば探索を止める。
- 仕様矛盾・失敗・直接依存の拡大があれば、該当する部分だけ探索を再開する。
- 必須テストの省略を「コンテキスト削減」と呼ばない。`0 tests`を成功根拠にしない。
- **現時点はコード未実装**。コードテスト欄が空の場合、実行したと偽らず未実装／未確認と書く。
