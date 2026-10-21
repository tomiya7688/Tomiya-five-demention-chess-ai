# Context Routing — 作業別の最小Working Set

[ObaPick](https://github.com/tomiya7688/ai-context-reducer)の「検索して候補を絞る」「必要な原典だけ読む」「完了条件で探索を止める」「変更内容に合わせて検証する」を実装するための**手動ルーティング表**。

**現状**：このリポジトリは設計段階。下表の `engine/` 等は**今後の予定ディレクトリ**で、まだ存在しない。実装されたときに対象ファイル名と検証コマンドを確定・更新する。存在しないコードやツールは実行可能とみなさない。

## 標準手順

1. **Taskを先に決める**：Goal / Required / Acceptance / Out of Scope（Issueの該当部分だけ）。
2. **Routeを一つ選ぶ**：関連する予定source／matching tests／契約を選択する。必要なら[Context Pack](context-pack-template.md)へ記録。
3. **Search first, read second**：リポジトリ内でパス・シンボルを検索し、対象と直接依存だけ読む。大きなファイルは関数／見出し周辺から読む。
4. **差分を優先**：共同作業時は更新履歴の全文よりchanged filenames、diff stat、関連commit概要を先に読む。仕様・コード本文が必要なら戻る。
5. **停止**：必要な事実とAcceptanceを確かめる手段が明確なら、追加の広い検索を止める。
6. **局所的に検証**：対象テストやfixtureを実行。Core共有契約、public API、形式変更なら範囲を広げる。
7. **短く報告**：変更、検証対象と件数、結果、未検証を記録。成功ログ全文は貼らない。

## ルーティング表

| Route | 最初の参照／今後のSource | 予定Matching tests | 必要な契約 | 検証の入口 |
| --- | --- | --- | --- | --- |
| `rules` | `doc/rules.md`, 将来 `engine/` と `learning_core/` のルール | [Core互換テスト](core-parity-tests.md) | 4軸移動・分岐・Present・Action/Submit | fixture→両Coreの回帰→差分 |
| `state` | `doc/core-contract.md`, 将来 `engine/` の時刻・世界線state | serialize/deserialize、make/unmake、歴史 | 半手phase・canonical state schema | 局所state test→両Core差分 |
| `runtime` | `doc/core-contract.md`, 将来 `runtime/` | 部分Move/Submitの拒否、正本不変、時計、Headless | Game/Player／Action contract | targeted runtime→固定seed対局 |
| `learning-core` | `doc/core-contract.md`, 将来 `learning_core/` | 共通ルールfixture、undo、状態往復 | Move/Action/Submit | 両Coreの対象testと差分test |
| `parity` | `doc/core-parity-tests.md`, `doc/rules.md`, 両Core | R01～R20、最初の相違Move | 同じprofile／入力／操作と結果 | fixture→差分→再生 |
| `search` | `doc/ai-model-roadmap.md`, 将来 `engines/` | 固定局面、探索制限・比較 | 合法Action／Submitと評価API | 再現実験→探索benchmark |
| `model-package` | `doc/ai-model-roadmap.md`, 将来 `ai_package/`, `models/` | manifest/asset/metadataの互換・負例 | `kadoka.ai_metadata.v1`（採用予定） | schema/実package→推論smoke |
| `creator-data` | `doc/ai-model-roadmap.md`, 将来 `tools/` | 学習Recipe、provenance、再実行 | Runtime／Creator境界 | 小さなdataset→headless比較 |
| `protocol` | `doc/architecture.md`, 将来 `protocol/` | JSON/notation roundtrip、外部接続 | 意味的Action／Game state | producer＋consumer smoke |
| `build-ci` | 将来 `CMakeLists.txt`, `.github/workflows/`, `tools/` | build／CTest／artifact起動 | 共通依存方向と配布境界 | syntax/policy→build→CTest→成果物smoke |
| `docs` | `README.md`, `AGENTS.md`, `AI_CONTEXT.md`, `doc/` の変更箇所 | リンク／相互整合性の確認 | 日本語正本・実装状況 | 該当リンクと参照先の存在確認 |

**ルートの追加条件**：異なるtaskで繰り返し効果がある場合に追加する。単発作業のために巨大な索引やツールを作らない。

## 探索と検証を広げるトリガー

- 時間／世界線／ターン／王手・終局の**ルール意味論**を変更した。
- CoreまたはPlayerの公開API、state schema、モデルpackage契約が変更された。
- Runtime／Creatorの依存方向やCMake targetを変更した。
- sourceとspecが矛盾する、未知の利用側が見つかる、局所テストが失敗する。
- 作った実行ファイル／配布物の動作そのものがAcceptanceである。

広げるときも原因・利用側・再現に必要な範囲を特定する。**「念のためrepo全体を読む」は既定の手順にしない。**

## Evidenceと停止条件

`Required evidence` をTaskごとに短く列挙し、`Known / Missing / Conflicts / Unverified` を必要に応じて記録する。`Missing = none`で矛盾が解消し、Acceptance判断が可能なら追加探索を終了する。新しいread/searchが何の不足を埋めるのか言えない場合も探索停止を優先する。

**強制的なファイル数やトークン数の上限ではない。** 必須のルール根拠、仕様、テストを省くことはしない。`0 tests`、空走査、無関係な成功結果は完了の証拠にしない。

## 参照

- [AI Context Reducer（ObaPick）](https://github.com/tomiya7688/ai-context-reducer)
- [変更に必要なファイルから読む](https://github.com/tomiya7688/ai-context-reducer/blob/main/docs/jp/変更に必要なファイルから読む.md)
- [調査を止める条件を決める](https://github.com/tomiya7688/ai-context-reducer/blob/main/docs/jp/調査を止める条件を決める.md)
- [変更に合わせて確認方法を選ぶ](https://github.com/tomiya7688/ai-context-reducer/blob/main/docs/jp/変更に合わせて確認方法を選ぶ.md)
