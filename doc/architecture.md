# アーキテクチャ（初期設計）

**状態：提案段階。** この文書はモジュール境界と検証上の不変条件を定める。具体的なAPI、データ形式、最適化方式は、ルールfixtureを確定してから設計する。

## 基本原則

Kadoka Shougi AI／Kadoka Othello AIに倣う。

1. **Standalone Core**：AIがなくてもゲームの局面作成、進行、判定、履歴の再現ができる。
2. **Authoritative state**：盤面／時間／世界線／ターン／終局はCoreだけが更新できる。
3. **Semantic action**：Human／Native AI／外部AIは同一の意味的操作境界を使う。AIは合法手を「決定」せず提案する。
4. **Runtimeは軽量**：Creator／Training／Dataset変換／rich analysisは対局ホットパスから外す。
5. **Correctness first**：時間移動と分岐を含むルールfixtureの再現に合格してから探索を高速化する。
6. **二つのフルルールCore**：対局用Game Coreと学習用Learning Coreを別実装し、共通仕様・同一fixture・差分テストで動作同値性を確かめる。

## 責務と依存方向（予定）

```text
UI / CLI / External Player / AI → Game Runtime → engine/ (Game Core)
                                                authoritative state

Creator / Training / Self-play / AI Search → learning_core/ (Learning Core)
                     └── 差分検証・対局記録の再生 ─→ engine/ (Game Core)
```

Runtimeの対局正本は **engine/ のGame Core**。Learning Coreは独立した高速シミュレータであり、対局stateを直接書き換えない。Core → 特定AI、Game Runtime → Creator／Training、Game Core → Learning Coreの依存は**禁止**。UI・protocolが第2のルールエンジンになってはいけない。互換条件は [Core契約](core-contract.md) を参照。

| モジュール | 役割（すべて予定） |
| --- | --- |
| `engine/` | **Game Core**：対局用の完全なルール、canonical state、Action/Submit、王手・終局、履歴と拒否理由 |
| `learning_core/` | **Learning Core**：同じ完全ルールの独立実装。大量探索向け状態複製、make/unmake、差分更新、学習用binding |
| `runtime/` | 観測の提供、Player/AI呼び出し、制限、Action検証、棋譜・Headless対局 |
| `engines/` | 探索、評価、学習モデル推論、キャラクターの意思決定 |
| `ai_package/` | 観測→モデル固有入力の変換、モデル／backend adapter |
| `protocol/` | CLI、外部プロセス、記譜法・メッセージの変換 |
| `tools/` | AI Creator、学習、self-play dataset、解析、bench、変換 |
| `models/` | 軽量メタデータ／descriptor（巨大な学習済みweightは別管理） |
| `tests/` | 局面・分岐・ターン・合法性・契約・再現性の回帰fixture |

AI Runtimeはモデル実行に**不可欠**なloaderや軽量評価器を持てる。ただし、対局中に不要な学習管理、バッチ解析や可視化はCreator側に分離する。

## 5D状態モデル（意味的な要件）

具体的なstruct・ビット割当て・エンコーディングは未決。Coreが表すべき情報の候補は以下。

- **Board**：ある世界線のある時刻／手番における配置。駒種、陣営、キャスリング等の必要な盤面属性。
- **Timeline**：時刻に沿ったBoard群と親分岐への参照、active/inactive等の状態。
- **Global state**：全timeline、Present、各盤面の着手可否、現在の陣営／ターン、履歴。
- **Action**：盤面内移動、時間／世界線間移動、必要なターン操作を区別できる表現。
- **Turn**：必要な盤面への着手群とターン確定。単一moveと同一視しない。
- **Result**：局面／ターン受理、拒否理由、終局状態、再現に必要なイベント。

位置（board座標）・時刻・timeline識別子・手番を区別する。世界線のIDや並びの正規化、共有ボードの識別、同一局面ハッシュは**ルール確定後に仕様化**する。時間移動によって過去のboardを直接書き換えず、分岐後の正本を保持する。

### トランザクションと検証

```text
Canonical state + Player observation
  → Player / AI が候補の操作・ターンを提案
  → Core が盤面／世界線／局面／ターンの全制約を確認
  → 成功: 新しいcanonical stateとイベント
  → 失敗: canonical stateは不変、理由を返す
```

複数操作からなるターンの途中状態が必要な場合は、**未確定の作業状態**と**確定済み正本**を区別する。途中操作の差戻し、ターン確定条件、無効提案時の再試行上限はルールとRuntimeの境界で明記する。AIに対してCore内の可変参照を直接渡さない。

## Move / Action / Submitの契約

4軸の駒移動を生成するだけでは、5Dチェスの合法ターン判定にならない。**Moveはターン内の部分操作**で、同一プレイヤーの順序付きMove列（Action）を作り、PresentとRoyal pieceの安全性を満たしたときだけ**Submit**してターンを確定する。途中のMoveで一時的にCheckを受けることと、最終Submitの違法性は区別する。

- 半手ごとのBoard識別にはtimeline/turnのほか、白・黒のphaseが必要。
- playableからplayableへのジャンプは両側のBoardを進め、historicalへのジャンプは新timelineを作り過去を上書きしない。
- Inactive timelineはPresentを決めない。分岐順とActiveの再計算はstate遷移の一部。
- 分岐Moveの並べ替えは局面を変え得る。探索で同一結果のActionを減らしても意味論は保つ。
- 無効なMove/Submitは、コミット済みの対局stateを変更しない。

ルール詳細は [rules.md](rules.md)、2 Coreの共通操作・拒否・undo・観測状態は [core-contract.md](core-contract.md)、確認すべき境界条件は [core-parity-tests.md](core-parity-tests.md)。

## 性能設計の検証項目

世界線分岐で履歴を含むstateが増えるため、以下は**候補**としてベンチマークで採否を決める。

- 不変Boardの共有、copy-on-write、差分stateとmake/unmake
- 増分合法手／利き計算、時間軸・世界線を含むstate hashとtransposition cache
- 局面ではなく**合法なターン全体**を探索単位とする必要性の検証
- 多数盤面の候補操作を段階的に生成し、枝分かれ数を抑える探索
- 固定入力・seed・探索node上限での同条件比較
- 近似評価で取りこぼすルール上の制約をCoreで必ず再検証

評価モデルは時間／世界線の配置、Royal pieceの脅威、Present、active timelineの影響を含めて再設計する。2Dチェスの評価値、合法手キャッシュ、チェック判定をそのまま持ち込まない。

## 開発と検証

1. [ルール整理](rules.md)でvariant・出典・例・未決事項を確定する。
2. 固定fixtureを先に用意し、**Game Core**の状態／合法性／分岐／ターン／終局を検証する。
3. **Learning Core**を同じ仕様で実装し、毎Move・Submit・終局についてGame Coreと差分テストする。
4. Player adapter、Headless RuntimeとRandom baselineを作り、学習側の対局記録をGame Coreで再生する。
5. Evaluate／学習・高速化・packageに進み、CI・性能測定は正確性と分けて記録する。

現在はコードがないため、実装済みのクラス／コマンド／CIを本書では宣言しない。
