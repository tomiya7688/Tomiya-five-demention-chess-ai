# アーキテクチャ（初期設計）

**状態：提案段階。** この文書はモジュール境界と検証上の不変条件を定める。具体的なAPI、データ形式、最適化方式は、ルールfixtureを確定してから設計する。

## 基本原則

Kadoka Shougi AI／Kadoka Othello AIに倣う。

1. **Standalone Core**：AIがなくてもゲームの局面作成、進行、判定、履歴の再現ができる。
2. **Authoritative state**：盤面／時間／世界線／ターン／終局はCoreだけが更新できる。
3. **Semantic action**：Human／Native AI／外部AIは同一の意味的操作境界を使う。AIは合法手を「決定」せず提案する。
4. **Runtimeは軽量**：Creator／Training／Dataset変換／rich analysisは対局ホットパスから外す。
5. **Correctness first**：時間移動と分岐を含むルールfixtureの再現に合格してから探索を高速化する。

## 責務と依存方向（予定）

```text
UI / CLI / External adapter ─┐
Native AI / Model adapter ───┼→ Runtime / Game → Authoritative Core
Human Player ────────────────┘                     ↑
Creator / Training / Dataset / Benchmarks ─→ AI / Runtime（利用側）
```

依存の向きは **Creator → AI/Runtime → Core**。Core → 特定AI、Runtime → Creator／Trainingは**禁止**。UI・protocolが第2のルールエンジンになってはいけない。

| モジュール | 役割（すべて予定） |
| --- | --- |
| `engine/` | ルール版、canonical state、分岐・時間座標、移動生成、状態遷移、王手・終局、履歴 |
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
2. 固定fixtureを先に用意し、Coreの状態／合法性／分岐／ターン／終局を検証する。
3. Core非依存のPlayer adapterとHeadless Runtimeを作る。
4. Random／Evaluateを比較し、探索・学習・packageはその上に拡張する。
5. CIと性能測定は兄弟プロジェクトの方式を参考にし、実際の対象・件数・成果物を検証する。

現在はコードがないため、実装済みのクラス／コマンド／CIを本書では宣言しない。
