# Game Core / Learning Core — 共通ルール契約（設計案）

> 状態: **未実装・仕様策定中**。本書はGame CoreとLearning Coreを**別実装**する際の共通の意味論を定義する。C++の関数名はインターフェース候補であり、既存APIを示さない。

## 目的

対局用の**Game Core**と、大量シミュレーション用の**Learning Core**はどちらも5Dチェスの**全ルール**（移動・歴史・分岐・Present・ターン・王手・終局）を実装する。学習用を「合法手判定なしの近似シミュレータ」にしない。

同じルールprofile・初期state・順序付き操作に対して、**観測できる意味論を一致**させる。2系統が内部表現・探索用高速化を共有する必要はないが、テストを共有する。

| | Game Core | Learning Core |
| --- | --- | --- |
| 用途 | 実対局、人間・外部AIとの共通Game境界 | 高速自己対局、rollout、学習データ生成、探索 |
| 優先度 | 厳密な検証・明確な拒否理由・監査性 | 意味論維持の上で速度・メモリ・再現性 |
| 更新 | validated transaction、明示的な確定 | make/unmake、差分局面、copy-on-writeなどを許容 |
| 正本 | 実際のゲームのcanonical stateの**唯一のauthority** | 独立シミュレータ内部のstate。対局の正本を直接変更しない |
| 拡張 | 記録／リプレイ、Player/API連携 | batch、vectorization、並列探索、Python学習binding |
| 検証 | referenceとして受け入れテストに合格 | 同じ受け入れテスト＋Game Coreとのdifferential test |

Game Coreの誤りが学習Coreに伝染する可能性があるため、**外部ルール出典と小さな期待結果付きfixture**を別に保管する。

## 同一とすべき状態の意味論

概念的には以下の情報を同じ意味で扱う。JSONや実際のC++レイアウトはまだ未決。

- **RuleProfile**: Standard等のvariant／駒／初期局面／規則version。
- **BoardAddress**: 世界線ID、時間T、白・黒の半手phase。squareはfile/rank。
- **BoardState**: 駒配置、必要なcastling／en-passant／移動済み属性、分岐親・関連情報。
- **TimelineState**: 不変の歴史Board列、最終Playable board、起源と分岐順序、active/inactiveの決定要素。
- **MultiverseState**: すべての世界線、Present、動作するプレイヤー、コミット済み履歴。
- **PendingTurn**: **未Submitの順序付きMove列と暫定state**。確定済み対局状態とは区別。
- **Result**: 成功／拒否理由／生成世界線／正規化state／手番／終局結果。

**時間・世界線と手番の識別、Moveの順序、PendingTurnの差異を失った圧縮表現は不可。** 世界線番号の内部実装が異なっても、決定したcanonical serializationで比較できること。

## 共通の操作（概念的API）

```text
load_initial(rule_profile) -> State
import_state(serialized, rule_profile) -> State
export_state(state) -> canonical serialized state
begin_turn(state) -> PendingTurn
generate_submoves(pending) -> candidates
apply_submove(pending, move) -> accepted/rejected + next pending
can_submit(pending) -> result + reasons
submit_action(pending) -> accepted/rejected + next committed state
game_status(committed_state) -> ongoing/checkmate/stalemate/...
```

Learning Core側だけに `clone`, `make/unmake`, `batch_step`, `generate_actions` を追加してよい。ただし**元の全ルールとSubmit条件を飛ばさない**。 `generate_submoves` は幾何学的・局所的候補を返すことがあり、**その1手だけでターン合法とは限らない**。

外部AIの提案するActionは、最終的に**Game Coreの検証済みSubmit**を通す。 `Game Core -> Learning Core` のランタイム依存は作らない。Headless自己対局にLearning Coreを使う場合でも、学習成果を採用する前にGame Coreで再生・差分検証する。

## MoveとTurnのトランザクション

- **1 Moveは半手盤面の一部を進める操作**。物理移動／他Playableへのジャンプ／Historicalへの分岐で必要なBoardの増分が違う。
- **1 Actionは順序付きMove列**。分岐Moveは順序を勝手に並べ替えない。
- **1 Submitはプレイヤーの操作を確定**。Presentが相手色の盤面になり、自分のRoyal pieceを捕獲されないことが条件。
- 途中状態ではCheckが残っていてもよい。**Moveごとに自己王手を排除してActionを失わない**。
- 不正Move／Submit時は**コミット済みstateは不変**。PendingTurnを残すか破棄するかはAPI仕様で明示し、暗黙の部分コミットは禁止。
- Undoは途中Turnの操作を巻き戻し、commit後に履歴を壊さない。再演時は同じ世界線生成順・結果になる。

## 実装上の境界

```text
UI / External AI / Player ──→ Game Runtime ──→ Game Core
                                          (canonical authority)

Training / AI Creator / Self-play ──→ Learning Core
              │                           │
              └──── parity replay ────────→ Game Core（検証時）
```

Game CoreとLearning Coreは**共通のルールデータ・fixture／codec仕様**を参照できるが、互いの内部struct／可変stateを共有しない。共通部品のコード共有は性能・独立性を検討して判断。学習用Coreに直接外部Action検証を委ねない。

C++20を第一候補とし、Pythonは高速なLearning Coreへのbindingとtraining/analysisに用いる想定。二系統のソース配置候補は `game_core/`, `learning_core/`, `runtime/` とする（現在存在しない）。

## 互換性の合格条件

1. 同じrule_profile・fixtureを読み、正規化stateが同じ。
2. 同じMove候補の許可／拒否と次stateが同じ。
3. 同じ順序付きMove列で、各中間状態・生成世界線・Presentが一致。
4. 同じSubmitの可否／拒否理由カテゴリ・次の手番・終局が一致。
5. undo→redoとserialize→deserializeの往復が一致。
6. 同じ固定seed・探索制限のヘッドレスデータがGame Coreで再生できる。

詳しいケースとテストレベルは [core-parity-tests.md](core-parity-tests.md) を参照。仕様の原典は [rules.md](rules.md)。**本書にあるテスト・APIはすべて未実装**。
