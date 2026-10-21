# Kadoka AI／モデル移植ロードマップ

**状態：計画。** 本リポジトリでは以下のAI・モデル・Toolingは未実装。兄弟プロジェクトの名前・方針を参考にし、5D Chess向けの仕様と実験で採否を決める。

## 兄弟プロジェクトとの対応

| 系列／手法 | 主な参照元 | 5D Chessでの予定 | 導入の前提 |
| --- | --- | --- | --- |
| Random / Greedy baseline | Shougi / Othello | 合法Action・Turnの候補生成と単純な評価 | Coreと固定fixture |
| Evaluate | [Shougi](https://github.com/tomiya7688/Kadoka-shougi-ai) | 手作り評価、反復深化、枝刈り | 正確な合法ターンと評価API |
| Evaluate Learning | Shougi | 手作り特徴＋学習済み評価／policy/value | Dataset・評価再現性 |
| Hyper Fast | Shougi | 低遅延、差分更新、候補削減 | hot-path benchmark |
| Mine Learning | Shougi | 系列独自の学習表現の検討 | input/targetの明確化 |
| Mine Learning Path | Shougi | 歴史・経路に依存する学習の実験 | timeline historyの正規化 |
| Tree AI | Shougi | 木探索／木構造の学習データ | tree reuse／探索評価 |
| Kadoka Best AI | Shougi | 最良の実証済み手法を統合 | ベースライン対戦比較 |
| Super Star | Shougi | 複数モデル／探索器のアンサンブル | 共通制限と公平な比較 |
| Obake Kadoka / Maru | Shougi / Othello | 個性のある弱い／特殊なAI | Core検証、名前・素材の権利確認 |
| Script / native evaluator | [Othello](https://github.com/tomiya7688/Kadoka-othello-AI) | evaluator差し替え、計測とinspection | 安全なRuntime境界 |
| AI Creator / Dataset / League | Othello | 解析、model比較、self-play、学習、評価リーグ | 独立した軽量Runtime |

上の「Shougi」欄は**機能名・設計の出典**を示し、参照元または本リポジトリにすべて完成しているという主張ではない。学習実験の派生モデルも、同じ基礎APIを確かめてから追加する。

## 実装順とGate

### Stage A — 同じ完全ルールを持つ二つのCore

- [ ] [ルール仕様](rules.md)のStandard候補、4軸移動、特殊手、Present、Action/Submitを実機fixtureで確定
- [ ] [Game Core](core-contract.md)：歴史が不変の世界線、部分Move／Submit、王手・終局を実装
- [ ] **Learning Coreも同一の完全ルール**を独立実装（簡略化した擬似ゲームにはしない）
- [ ] [R01～R20のfixtureと差分テスト](core-parity-tests.md)で各Move・Submit・生成世界線・終局を照合
- [ ] Human / Randomと固定seedのHeadless対局、学習側棋譜のGame Coreによる再生

**Stage A合格前に学習したモデルを対局用モデルとして採用しない。**

### Stage B — 評価／探索

- [ ] Evaluate（time/timeline awareな特徴量）
- [ ] 固定探索node・時間制限／反復深化
- [ ] 探索失敗、打ち切り時の有効なproposal
- [ ] 同条件での速度・探索量・勝率比較

### Stage C — 学習／データとCreator

- [ ] Datasetの記録・出典・生成条件・train/validation分離
- [ ] Evaluate Learning、Mine Learning（Path含む）、Tree AI
- [ ] モデル比較、複数AI再評価、confidence／uncertaintyの研究
- [ ] 形式・asset validation、信頼できない実行モジュールの扱い

### Stage D — 統合／実験的モデル

- [ ] Hyper Fastの増分最適化を計測で検証
- [ ] Kadoka Best AI / Super Star
- [ ] Obake Kadoka / Maru（合法性を破らない）
- [ ] ベンチマーク・AI League・成果物smokeで比較

## 5D Chess専用の探索研究

- **Global positionを評価**：複数時刻／世界線、PresentとRoyal pieceへの脅威を考慮する。
- **ターン単位の分岐**：単一のMoveだけではターン確定を保証しない。**順序付きMove列＝Action**を構成し、PresentとRoyal pieceの安全性を満たすSubmitまで探索する。
- **世界線の増殖**：状態共有／差分・枝刈り・段階的Move生成を検証する。historical board着地時の分岐とPlayable間ジャンプの両Board進行を区別する。
- **Action順序・正規化**：非分岐Moveの同一結果の重複を抑える一方、分岐Moveの相対順序は保つ。canonical state比較と探索hashを分離する。
- **時間・世界線ハッシュ**：類似局面と同一局面を区別するため、タイムライン構造も含む正規化を検討する。
- **探索予算**：時間、node、メモリ、worldline数、モデル推論回数の上限を記録して再現する。
- **研究評価**：固定局面セット、相手AI／先後／seedをそろえて比較する。Learning Coreの対局データはGame Coreでも再生・整合性を確認する。

通常チェス用の駒価値や探索アルゴリズムはベースラインとして参考にできるが、正確性・性能を実測せずに「移植済み」としない。

## モデル／Packageの共通化方針

兄弟シリーズの `kadoka.ai_metadata.v1` は**共通のモデル識別・実験情報**のための契約であり、探索内部の重み形式ではない。

導入予定の必須情報：`format`, `model_id`, `model_name`, `model_version`, `architecture`, `game`, `license`。

追加候補：`search_config`, `runtime_requirements`, `determinism`, `training_recipe`, `dataset_provenance`, `benchmark_results`, `source`, `distribution`。

`game` の5D用識別子は**兄弟契約との整合を確認してから確定**する。例として `five_d_chess` が考えられるが、まだ正式な共通値ではない。

Othelloの `manifest.json`（パッケージ入口）＋`model.json`（`kadoka.model.v1` asset記述子）の考え方を参考にする。将来のassetには evaluator、NN、search config、book、tree data等を許容し、未知のrequired assetはload失敗とする設計を検討する。**この形式を実装済みと誤認しない**。ゲームCoreはmodel packageを知らない。

学習・解析・変換・rich inspectionはCreatorへ、対局時に必要な推論はRuntimeへ配置する。未信頼なscriptやnative libraryを無条件に同一プロセスで実行しない。

## 実験・ライセンスの約束

- model・重み・Datasetは由来、使用条件、version、学習設定を分離して追跡する。
- 巨大weight／DatasetはGitに無条件でcommitしない。
- 技術AI／学習重みと、Kadoka／Maru等の名前・キャラクター画像の権利を区別する。
- 兄弟プロジェクトからのコード・重み・asset移植は、それぞれのライセンス・再配布条件を確認する。
- テスト・性能計測・学習を行っていない段階では結果を推測で記載しない。
