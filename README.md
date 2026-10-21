# Kadoka 5D Chess AI

**時間移動と世界線分岐を扱うチェスAIを、Kadoka AIシリーズの設計思想・実験基盤で作るプロジェクト。**

> 「そんなんやってるのお前だけだよ」シリーズ：五次元チェスAI編

**開発状況：初期設計フェーズ。** 現在はREADMEと開発指針を整備する段階です。ゲームCore、AI、ビルド、テストは**未実装**です。下記の構成・モデルは、明記がない限り実装予定を示します。

## 目標

- 時間軸・分岐世界線を含む局面を正確に扱える**独立したゲームCore**を作る。
- 人間・ランダムAI・探索AI・学習AI・外部AIを**同じ対局インターフェース**で差し替える。
- [Kadoka Shougi AI](https://github.com/tomiya7688/Kadoka-shougi-ai)の探索・評価・AIファミリーと、[Kadoka Othello AI](https://github.com/tomiya7688/Kadoka-othello-AI)のRuntime／Creator分離・モデル／パッケージ・研究基盤を段階的に取り入れる。
- 強さだけでなく、正確性、再現性、推論時間、メモリ使用量、実験の比較しやすさを重視する。
- [AI Context Reducer（ObaPick）](https://github.com/tomiya7688/ai-context-reducer)の手法を、AI支援開発の標準手順として運用する。

## ルールの対象と注意点

想定する参考ルールは「[5D Chess With Multiverse Time Travel](https://5dchess.wiki.gg/wiki/Rules)」です。ただし、**このゲームとの完全互換・初期配置・バリアント・記譜法は未確定**であり、実装前に検証用局面を含めて固定します。

通常のチェス盤だけでなく、盤面座標（file/rank）、時刻、世界線、各盤面の手番、過去への着手による分岐、active/inactive timeline、Present、王手と終局を扱う必要があります。特に**1駒の移動＝対局の1ターン完了とは限りません**。複数盤面への着手とターン確定を区別します。

曖昧な規則を推測で実装済み仕様にせず、未決事項・出典・テストケースを [ルール整理](doc/rules.md) に記録します。商用ゲームの公式実装・公式AIであることを意味しません。

## Kadokaシリーズから引き継ぐ設計

| 方針 | 本プロジェクトでの扱い |
| --- | --- |
| **Authoritative Core** | 盤面・世界線・合法手・ターン・終局の正本はCoreだけが保持する |
| **AIは提案者** | AIが返す手／ターン操作をCoreが検証し、違法な提案で正本状態を壊さない |
| **Core単独動作** | AIなしで局面作成・合法性判定・棋譜再生・人間同士の進行ができる |
| **Runtime / Creator分離** | 対局に必要な軽い推論経路と、学習・変換・解析・レポートを分ける |
| **差し替え可能なPlayer/Engine** | Random、評価関数、学習、外部プロセスなどを共通の意味的Action境界に接続する |
| **再現可能な実験** | 局面・乱数seed・探索制限・モデルversion・ルールversionを記録する |
| **Correctness before strength** | 強いAIより先にルールと状態遷移、undo／分岐／終局の回帰テストを完成させる |
| **Headless first** | GUIに依存せず対局・検証・自己対局・性能計測を行う |

兄弟プロジェクトの実装をそのまま移植するのではなく、**設計・検証方法・共通メタデータは流用し、5D固有のルールと探索は専用実装**にします。詳細は [アーキテクチャ](doc/architecture.md)。

## 実装予定のAI／モデルファミリー

Kadoka Shougi AIの計画にあるファミリーを起点に、Othello AIの評価器・パッケージ方式を組み合わせます。**以下はすべて本リポジトリでは未実装**です。

| ファミリー | 5Dチェスでの計画 |
| --- | --- |
| Random / Greedy（基準） | 合法な着手・ターン構成と簡単な評価を検証するベースライン |
| **Evaluate** | 手作り評価関数＋反復深化、枝刈り、局面探索 |
| **Evaluate Learning** | 学習済み評価・policy/valueと探索のハイブリッド |
| **Hyper Fast** | 差分更新、候補絞り込み、時間予算を重視する軽量AI |
| **Mine Learning / Mine Learning Path** | 系列独自の学習・履歴／経路特徴量を用いる実験 |
| **Tree AI** | 木探索・木構造の学習データを用いた実験 |
| **Kadoka Best AI** | 実測で有効だった手法を組み合わせる実用型 |
| **Super Star** | 複数評価器・探索器を組み合わせる大型実験 |
| **Obake Kadoka / Maru** | 強さ以外の個性を持つキャラクターAI（合法性はCoreが保証） |

5D特有の**ターン全体の合法性、世界線増殖、過去局面参照、巨大な分岐係数**に合わせて探索単位・状態共有・キャッシュ・評価関数を再設計します。通常の2Dチェス用Alpha-Betaや駒価値を無条件に流用しません。

将来の共通モデルメタデータは兄弟シリーズの `kadoka.ai_metadata.v1` に合わせ、モデル識別子・architecture・game・version・license・seed・学習データ由来・benchmark条件を扱う方針です。モデル本体の形式、互換性、ゲーム識別子は本プロジェクトで仕様確定してから実装します。

→ [AI／モデル移植ロードマップ](doc/ai-model-roadmap.md)

## 構成案

```text
engine/       # [予定] 5D局面・世界線・合法手・ターン確定・終局の正本
runtime/      # [予定] Game/Player・時間制限・合法性検証・Headless対局
engines/      # [予定] 評価・探索・学習モデル・キャラクターAI
ai_package/   # [予定] モデルロード・外部AI接続・形式変換境界
protocol/     # [予定] CLI / 外部プロトコル / 記譜法adapter
tools/        # [予定] AI Creator、学習、self-play、benchmark、検証
models/       # [予定] メタデータ、manifest、軽量な例（巨大weightは別管理）
tests/        # [予定] ルールfixture、回帰テスト、再現可能な対局
doc/          # 現時点の仕様・設計・コンテキスト経路
AI_CONTEXT.md # AI開発の最小入口
AGENTS.md     # エージェント共通の短い作業規約
```

言語は**C++20（Core／Runtime／高頻度探索）＋必要に応じてPython（学習／解析ツール）**を第一候補とします。CMake／CTest、Linux／Windows CI、固定seedのHeadlessテストは、実装段階で兄弟プロジェクトの実証済み方式から取り入れます。**現時点でビルドコマンドは存在しません。**

## AI Context Reducer（ObaPick）を開発手順に採用

コンテキストを削ること自体ではなく、**必要な根拠を保ったまま無関係な情報を読ませないこと**が目的です。全ソース、全Issue、全doc、生成物を起動時に読み込みません。

1. **入口**：[`AGENTS.md`](AGENTS.md) → [`AI_CONTEXT.md`](AI_CONTEXT.md) → 現在のIssue／依頼。
2. **Route**：[`doc/context-routing.md`](doc/context-routing.md)から該当責務を選び、「ソース／対応テスト／必要な仕様／検証」を限定する。
3. **Search first, read second**：まずパス・シンボル・見出し・差分を検索し、該当範囲と直接依存だけ原典で確認する。
4. **Task capsule**：必要なら [Context Packテンプレート](doc/context-pack-template.md)に `Goal / Required / Acceptance / Out of Scope / Working Set / Validation` を短くまとめる。これは**索引であり原典の代替ではない**。
5. **停止条件**：目的、必要な根拠、完了条件、検証方法がそろったら広い調査を止める。矛盾・新しい影響・テスト失敗時のみ対象を広げる。
6. **検証のルーティング**：関連テストから始め、ルール／公開API／依存境界変更では広い検証へ。成功ログ全文は残さず、実行した検査・件数・結果・未検証範囲を短く報告する。

`build/`、生成データ・モデルweight、巨大な棋譜／ログ、関係ない履歴は通常読まない。**ルールの正確さや受け入れ条件の検証はコンテキスト削減のために省略しません。**

現段階のルーティングは**Markdownによる運用**です。兄弟プロジェクトの `context_route.py` 相当やObaPickの補助ツールを取り入れる場合は、導入効果を確認してから追加します。存在しないスクリプトを実行できるものとして案内しません。

参考：[ObaPickの基本手法](https://github.com/tomiya7688/ai-context-reducer/blob/main/docs/jp/コンテキスト削減の基本.md)／[Context Pack](https://github.com/tomiya7688/ai-context-reducer/blob/main/templates/CONTEXT_PACK.md)

## 開発順序と完了条件

1. **P0: ルール仕様とfixture** — 対象バリアント、時刻／世界線／Present／チェック／終局を根拠付きで確定する。
2. **P1: Core** — 不変条件、局面の保存・復元、時間移動・分岐、合法手とターン確定をテストする。
3. **P2: 対局基盤** — Human/Random、Headless、棋譜、再現可能なseed付き対局を完走させる。
4. **P3: 探索基盤** — 状態差分、make/unmake、ハッシュ、探索制限、perft相当・性能計測を導入する。
5. **P4: AIファミリー** — Evaluate→学習・高速化→Tree/Mine→統合・キャラクターAIの順に段階導入する。
6. **P5: Creator／学習／配布** — Dataset provenance、複数AI比較、リーグ、パッケージ検証、CI・配布物検証を追加する。

各段階は**仕様・実装・テスト・再現手順**がそろって初めて完了とし、完了していない段階を「対応済み」と記述しません。

## ドキュメント

- [AI_CONTEXT.md](AI_CONTEXT.md) — AIの小さい開始地点
- [AGENTS.md](AGENTS.md) — AIエージェント共通指示
- [doc/context-routing.md](doc/context-routing.md) — 作業別の読む場所・検証範囲
- [doc/context-pack-template.md](doc/context-pack-template.md) — タスク単位のコンテキスト整理
- [doc/architecture.md](doc/architecture.md) — Core／Runtime／Creatorと5D状態の設計
- [doc/rules.md](doc/rules.md) — ルール参照先と未決事項
- [doc/ai-model-roadmap.md](doc/ai-model-roadmap.md) — シリーズAI／モデル移植計画

## ライセンス

**本リポジトリのライセンスは未設定です。** 兄弟プロジェクトのソース・モデル・学習データ・キャラクター素材を利用する前に、それぞれの適用ライセンス、再配布条件、出典を確認します。Kadoka／Maru等の**キャラクター名・デザイン**と**AIアルゴリズム・学習済み重み**の利用条件は混同しません。
