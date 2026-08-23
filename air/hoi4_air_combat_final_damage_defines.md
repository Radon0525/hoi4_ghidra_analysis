# HOI4 `AIR_COMBAT_FINAL_DAMAGE_*` 3係数の解析結果

## 結論

対象の `hoi4.exe` では、次の3 define は **Lua define として登録・読込されるだけで、ゲームプレイ処理から参照されていない**。

```lua
AIR_COMBAT_FINAL_DAMAGE_SCALE = 0.015
AIR_COMBAT_FINAL_DAMAGE_PLANES = 50
AIR_COMBAT_FINAL_DAMAGE_PLANES_FACTOR = 0.1
```

したがって、この実行ファイルでは3値を変更しても、通常空戦の撃墜数、disruption、少数航空隊の丸め、空母航空隊同士の撃墜数には影響しない。

define コメントは旧実装の説明が残ったものと判断できる。

## Ghidra上の参照結果

| define | 値の格納先 | 参照元 | 結果 |
|---|---:|---|---|
| `AIR_COMBAT_FINAL_DAMAGE_SCALE` | `0x143246e20` | `0x1407b9ce0` (`FUN_1407b9b20`) | 登録時のアドレス取得1件のみ |
| `AIR_COMBAT_FINAL_DAMAGE_PLANES` | `0x143246eac` | `0x1407b98c0` (`FUN_1407b9700`) | 登録時のアドレス取得1件のみ |
| `AIR_COMBAT_FINAL_DAMAGE_PLANES_FACTOR` | `0x143246f30` | `0x1407b9ad0` (`FUN_1407b9910`) | 登録時のアドレス取得1件のみ |

3件とも、参照型は Lua 読込先を渡すための `LEA` だけだった。戦闘処理からの `READ`、値の比較、乗算、加算は存在しない。

登録関数にはビルド時ソース位置 `defines_military.h:0x1f0`、`:0x1f1`、`:0x1f2` が残っている。文字列アクセサも存在するが、define 名を返すメタデータ用途であり、値を戦闘計算へ渡すものではない。

## 使用中defineとの対照

実際に通常空戦の撃墜数へ使われる `COMBAT_DAMAGE_SCALE` の格納先 `0x14324b2a0` には、撃墜計算本体から次のREAD参照がある。

```text
0x140f4aa52  MOV RDX, [0x14324b2a0]
caller: FUN_140f4a790
```

空母航空隊同士の組合せで使われる `COMBAT_DAMAGE_SCALE_CARRIER` (`0x14324b398`) にも、同じ関数からREAD参照がある。

```text
0x140f4aa5c  CMOVNZ RDX, [0x14324b398]
caller: FUN_140f4a790
```

この対照から、単にGhidraがdefine参照を認識できていないのではなく、`AIR_COMBAT_FINAL_DAMAGE_*` 3項目だけが現行戦闘コードへ接続されていないことが分かる。

## コメントが意図していたと思われる旧仕様

名前とコメントからは、過去の実装で次の目的を持っていたと読める。

- `FINAL_DAMAGE_SCALE`: disruptionだけを受けた機体のうち、1戦闘で死亡へ変換できる割合の上限
- `FINAL_DAMAGE_PLANES`: 小規模航空隊で端数が消える問題を抑えるための基準機数
- `FINAL_DAMAGE_PLANES_FACTOR`: 上記の少数機補正に使う係数

ただし、対象exeにはこの3値を組み合わせる計算が残っていないため、旧式の正確な数式を現在のバイナリから復元することはできない。コメントから推測した式を現行仕様として扱うべきではない。

## 現行の撃墜処理との関係

通常空戦の撃墜数は `FUN_140f4a790 @ 0x140f4a790` で計算される。概略は次の流れである。

1. 参加機数と Air Attack から攻撃寄与値を作る。
2. `COMBAT_DAMAGE_STATS_MULTILPIER` を掛ける。
3. 速度優位・Agility劣位の補正を適用する。
4. Air Defenceで割り、`COMBAT_DAMAGE_SCALE` または `COMBAT_DAMAGE_SCALE_CARRIER` を掛ける。
5. 期待撃墜数の整数部を確定し、小数部を `0..99999` の乱数で確率的に丸める。
6. 交戦可能な防御機数で上限をかける。

少数航空隊の端数問題も、この現行経路では期待値の小数部を確率的に1機へ丸めることで処理される。`AIR_COMBAT_FINAL_DAMAGE_PLANES` と `AIR_COMBAT_FINAL_DAMAGE_PLANES_FACTOR` はこの丸め処理に入らない。

## 設定変更上の意味

- この3 define は残しても削っても、対象exeの戦闘結果は変化しないと考えられる。
- 空対空の致死率を調整したい場合は、実際にREADされる `COMBAT_DAMAGE_SCALE`、`COMBAT_DAMAGE_SCALE_CARRIER`、`COMBAT_DAMAGE_STATS_MULTILPIER` などを対象にする。
- disruptionそのものの調整は、別系統の `DISRUPTION_*` defineを対象にする。
- バニラと今回参照したMODでは3値が同一であり、現状はMOD差の原因にもなっていない。

## 確度

**高確度。** Ghidraの全参照検索で各格納先を直接確認し、使用中defineのREAD参照と比較した。残る一般的な留保は、自己書換えコードや文字列名による動的検索のような静的参照に現れない実装だが、同じ define システムの使用中項目が直接グローバルREADになっていることから、その可能性は低い。
