# 5D Core同値性・回帰テスト計画

> 状態: テスト**計画**。実装済みのfixture／CTest／CIはまだない。

[ルール](rules.md)から独立に作る「出典と期待値付きゴールデンfixture」を**Game Core／Learning Coreの両方に投入**する。Game Coreの出力を唯一の期待値にすると、そのバグを二系統が共有するため、公開ルールや公式ゲームで確かめたケースを別途必要とする。

## 層別テスト

| レベル | 対象 | 成功条件 |
| --- | --- | --- |
| L0: ルールfixture | 個別の駒移動・時間ジャンプ・分岐・ターン | 原典で確認した期待state／accept/rejectに一致 |
| L1: Core内部 | Game／Learningそれぞれのstate・apply・undo・Submit | 自分の状態不変条件と履歴・往復が成立 |
| L2: 2 Core差分 | 同一state＋順序付きMove列 | 毎Move後とSubmit後の**正規化state**、結果が一致 |
| L3: Headless | 固定seed・行動列／探索制限 | Game CoreでLearning Core出力を再生可能 |
| L4: 性能 | 同じ局面／ハードウェア／制限 | throughput、peak memory、分岐数の測定（正確性とは別評価） |

## 最低限のfixture項目

実験用のJSON／別形式は正式なschema決定後に作成し、各ケースに `case_id`, `rule_profile`, `source`, `verification_status`, `initial_state`, `moves`, `submit_request`, `expected` を記録する。`verified`／`provisional`／`unknown` を区別し、**未確認値を正解ラベルにしない**。

| ID | 対象 | 見るもの |
| --- | --- | --- |
| R01 | Standard開始と時間の半手phase | 初期Boardの識別、T／色、権利 |
| R02 | 盤面内Rook／Bishop／Queen／King／Knight | ベクトル、同色駒、遮蔽、王手関連 |
| R03 | 4軸のUnicorn／Dragon | 3軸／4軸の距離、通過盤面 |
| R04 | Knightの時間ジャンプ | 間の駒・存在しないBoardを飛び越す |
| R05 | Pawnの世界線方向／捕獲 | 前進、2歩、XY／TL、混合方向の禁止 |
| R06 | Castling／en passant／Promotion | 未移動属性、王手経路、特例とQueen昇格 |
| R07 | 同じPlayable boardでの物理Move | 履歴が不変、次の半手Board |
| R08 | 別Playable boardへの非分岐ジャンプ | 出発元・到着先の両方が進む |
| R09 | Historical boardへのジャンプ | 新世界線、分岐元、旧履歴不変 |
| R10 | Active／Inactiveの境界 | n番目の活性化、再活性化 |
| R11 | Presentの移動と任意盤面 | 必須のMove、余分なMove、Submit可否 |
| R12 | 複数MoveのAction | 途中Check許容、最終Submitの検証 |
| R13 | 時間越しのRoyalへの脅威 | 捕獲可能性、Checkの場所 |
| R14 | Checkmate／Stalemate | 合法Actionの存在性、チェックの区別 |
| R15 | 分岐Actionの順序 | 分岐順を変えると結果が変わるケース |
| R16 | 非分岐Moveの標準順序 | 同一の最終stateに正規化できるケース |
| R17 | 不正Actionの拒否 | コミット済みstate不変、拒否理由 |
| R18 | serialize／deserialize、clone／undo | 状態同値性、履歴・権利の保全 |
| R19 | Game Core対Learning Core | 各部分Move・Present・Submit結果の差分 |
| R20 | 学習データの再生 | 固定seed／モデルversion・操作列と対局再現 |

R01～R20は**テスト計画のID**であり、実測結果や期待値の記入済みfixtureを意味しない。

## 差分テストの流れ（予定）

```text
verified fixture / reproducible state + rule_profile
     ├── Game Core: import → begin → apply moves → submit → export
     └── Learning Core: import → begin → apply moves → submit → export
                             ↓
   compare normalized boards, timeline lineage, present,
   active/inactive, pending moves, submit legality, result
                             ↓
    mismatch: retain seed, case, first differing move and fields
```

- 成功時はケース数・ルール版・結果のみ表示し、大量の盤面全文は残さない。
- 失敗時は最初に相違したMoveと状態fieldに絞って差分を出す。
- 学習Coreの差分state／transposition hashは**等価性の証拠ではない**。canonical exportを比較する。
- すべての可能なActionの列挙は現実的でないことがある。小さな局面では全列挙、大きな局面では固定seed／制限付きの候補集合と既知の詰みfixtureを使い、未検証範囲を明記する。
- 公開APIやrule_profileを変えた場合はL0～L3を再確認。局所最適化は対象L1/L2から始めるが、採用前にL3も確認する。
- テスト0件、失敗を無視した実行、固定fixture未照合は「正しい」と報告しない。

## 開発順

1. 正本ルールprofileとBoard／Action形式を合意する。
2. **出典付きの小さなfixture**（最初はR01,R02,R07,R09,R11,R12）に実際の初期state・操作・期待値を付ける。
3. Game Coreを作りL0/L1を通す。
4. Learning Coreを独立実装しL0/L2を通す。
5. Headless自己対局でL3を通し、そこで初めてL4の性能最適化を採用する。

参考： [5D Wiki Rules](https://5dchess.wiki.gg/wiki/Rules)、[Action](https://5dchess.wiki.gg/wiki/Action)、[Programming](https://5dchess.wiki.gg/wiki/Programming)。
