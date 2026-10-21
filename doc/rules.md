# 5Dチェス ルール調査・実装仕様の入口

> 調査更新: 2026-10-22 / 状態: 公開資料を照合した**暫定仕様**。公式ゲーム実機とのfixture照合・完全互換は未確認。

対象候補は「5D Chess With Multiverse Time Travel」の **Standard（8×8、初期世界線1本）**。Turn Zero、盤面サイズ違い、特殊駒入りなどは将来の独立した `rule_profile` とする。Standard採用は開発上の第一候補で、互換対象の正式決定までは固定しない。

## 情報源と確認区分

- [5D Chess Wiki: Rules](https://5dchess.wiki.gg/wiki/Rules) — 駒の移動、分岐、active timeline、checkmate。
- [5D Chess Wiki: Action](https://5dchess.wiki.gg/wiki/Action) — moveとactionの違い、標準順序、合法action探索。
- [5D Chess Wiki: Glossary](https://5dchess.wiki.gg/wiki/Glossary_of_Terms) — Present、Submit、Playable、Moveなどの用語。
- [5D Chess Wiki: Programming](https://5dchess.wiki.gg/wiki/Programming) — position／board／multiverseの表現例。
- [5D Chess Wiki: Notation](https://5dchess.wiki.gg/wiki/Notation) — Shad式表記、白側＋L／黒側−Lの表記。
- [開発者によるCastlingの意図した仕様](https://steamcommunity.com/app/1349230/discussions/0/2791621875939533777/) — 2020年8月の説明。

以下の「公開資料による確認」は**参照ルールの確認**であり、対局ソフトで同じ挙動をテストした意味ではない。

## 座標・盤面・歴史

駒の移動は四つの軸 `(file=x, rank=y, turn=T, timeline=L)` で決まる。加えて同じ `(T,L)` に「白が動かす盤面」と「黒が動かす盤面」の2枚が対応するので、**Board識別子には手番／半手（phase）の区別が必要**。外部表記の `+L/-L` は Shad式では白／黒が生成した世界線を示す。

- `playable board`: その世界線の最新盤面。**この盤面が自分の色のとき**駒を動かせる。
- `historical board`: 既に後続の盤面が存在する過去の盤面。駒の到着先にはなれるが、その盤面を上書きしない。
- 同じ盤面内の移動は、その世界線に**次の半手盤面**を生成する。
- ある `playable board` から別の `playable board` へ移動する場合、**出発元と到着先の両盤面を進める**。出発元から駒を除去し、到着先へ配置する。
- `historical board` へ飛ぶ場合、**新しい世界線を分岐**する。出発元も進み、既存の歴史は変更されない。
- 時間／世界線間の移動では、着手側と反対色の半手盤面は移動判定の対象にしない。

Board履歴は意味論上不変。実装上の永続データ構造・差分更新／undoは同一の結果を保証する限り自由とする。

## 駒の動き（公開資料による確認）

差分 `(dx,dy,dT,dL)` の**非ゼロの軸**数と距離を基準にする。

| 駒 | 幾何学的な条件 |
| --- | --- |
| Rook | 非ゼロ軸が1本。任意距離で経路の遮蔽を確認 |
| Bishop | 非ゼロ軸が2本。各絶対値が等しい。遮蔽確認 |
| Unicorn | 非ゼロ軸が3本。各絶対値が等しい。遮蔽確認 |
| Dragon | 非ゼロ軸が4本。各絶対値が等しい。遮蔽確認 |
| Queen | 1～4本の非ゼロ軸。各絶対値が等しい。遮蔽確認 |
| King | Queenと同方向で各軸1マス。Royal piece |
| Knight | 異なる2軸にそれぞれ長さ2、1。中間の駒・欠損した盤面を跳ぶ |
| Pawn | 縦の盤面方向または世界線方向の**いずれか1軸**に前進。初回は条件付き2マス。捕獲は `(x,y)` または `(T,L)` の前方斜め |

Pawnの「前方」は各プレイヤーの相手陣側と、**相手が世界線を生成する方向**を指す。Pawnは空間・非空間を混ぜた斜め捕獲はできない（Brawnは別ルール）。滑走駒は途中の盤面／マスで遮蔽を判定し、Knightは途中の盤面が欠けていても跳べる。

### 特殊手とvariant

- **Castling** は同じ2D盤面上の操作。開発者の説明ではKingの現在位置または通過マスが**その盤面上の駒**から攻撃されていると不可。最終的な5D全体の王手判定はターンSubmit時に別途行う。
- **En passant** は同じ盤面内に限定され、時間や別世界線への拡張はない。
- **Promotion** は通常のPawnが最終rankに到達すると**Queenのみ**へ昇格（アンダープロモーションなし）。
- Unicorn、Dragon、Brawn、Princess、Common King、Royal Queen等は種類として存在するが、**Standard初期局面に全て配置するわけではない**。variant別に定義する。

Castling権利・en passant状態を過去への分岐でどう引き継ぐかは、盤面属性として保持したうえで実機fixtureで確認する。

## Active／InactiveとPresent

各陣営が作った世界線の**n本目**は、相手の生成数が少なくとも `n-1` 本ならActive。それ以外はInactive。原初世界線はActive。相手の新分岐により以前Inactiveだった世界線がActiveに戻ることがある。

**Present** は全Active世界線の最新盤面のうち、**最も古い時間位置**にあるものを指す。Inactive世界線はPresentを決めない。

プレイヤーは自分の番に複数の盤面を動かせる。Presentが**相手色の盤面**に移るまでの着手は必要だが、未来にある自分のplayable boardやInactive世界線での着手は常に必須とは限らない。途中の分岐・Activeの変化で必須対象も変化する。**1 Move = 1 Turn ではない**。

## Move、Action、Submit、勝敗

- `Move`: 駒1個の物理移動／他盤面へのジャンプ。**ターン途中の状態**は一時的にチェックを受けてもよい。
- `Action`/`Moveset`: 同じプレイヤーの**順序付き**Move列で、完了時に合法なSubmitができるもの。
- `Submit`: Presentが相手の色になり、**自分の全Royal pieceの捕獲可能性**が解消した場合のみ成功し、相手へ手番を渡す。
- `Check`: 仮に現在Present上の必須盤面をパスしたとすると、相手がRoyal pieceを捕獲できる脅威。
- `Checkmate`: 自分のRoyal pieceがCheckされ、**合法Actionが1つもない**。
- `Stalemate`: Checkではないが合法Actionが1つもない（引き分け）。

**単一Moveの自己王手フィルタで合法Actionを判定しない。** 出来上がったターン全体の存在を調べる必要がある。

## Actionの順序と重複

[Action](https://5dchess.wiki.gg/wiki/Action) によれば、物理Move／非分岐ジャンプは同じ結果になる順序の入れ替えが可能。一方、**分岐ジャンプの順序は世界線番号などの結果に影響する**。

探索や学習では、非分岐Moveを世界線順に並べ、その後に分岐Moveを**元の相対順を保持して**置く標準順序を、同一結果の二重列挙削減策として採用候補にする。**仕様上の全候補の保存**と**探索での等価類圧縮**は別責務。

## 未確認と次の工程

- Standardの具体的な初期Board・T0/T1のエンコーディング、初期権利を**独立fixture**に固定。
- 半手盤面のID、分岐後のWorldline番号、複数同ターン分岐の割当て。
- Castling／en passant／Promotionの時間移動後の細かい境界条件。
- 実機とのCheck・stalemate・action判定の突き合わせ。
- 大量世界線、異なるvariant、特殊駒の細部、版依存の差異。

**公開資料を読んだだけの状態では完全互換を宣言しない。** 計画するテストは [Core互換テスト](core-parity-tests.md)、APIと二系統Coreの関係は [Core契約](core-contract.md) を参照。
