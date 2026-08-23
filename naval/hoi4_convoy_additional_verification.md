# HOI4 輸送船――既存ログとの差分精密検証

対象は、提供された HOI4 1.17 系 Ghidra プロジェクトと Workshop `820039020`（innovative balance v13.00）である。

この文書は、参照タスク `019fff9b-6fd2-72e3-8a86-31731975ba3b` ですでに確定している次の内容を繰り返さない。

- 輸送船撃沈後の航路効率低下と回復
- 船団襲撃任務の探索・追跡進捗
- 船団護衛任務の効率と護衛参加
- 輸送船の固定被弾プロファイル `CONVOY_HIT_PROFILE = 120`

以下は、新たに確定した事項、または既存ログで中確度だった箇所の訂正だけである。

## 1. 最重要の訂正――通常輸送船の戦闘参加隻数

既存の船団襲撃ログ §8.1 にある概略式は、係数の対応が逆だった。`FUN_140f6ec10 @ 0x140f6ec10` と各 define のロード先を照合すると、実装は次になる。

### 1.1 変数

```text
S = 襲撃任務部隊の実効水上探知集計値
u = 0以上1未満の乱数
R = 航路が持つ海域数
C = 対象航路の輸送船数

A = COMBAT_DETECTED_CONVOYS_FROM_SURFACE_DETECTION_STAT = 0.10
B = CONVOY_ATTACK_BASE_FACTOR                           = 0.15
K = CONVOY_ROUTE_SIZE_CONVOY_SCALE                     = 0.01
```

`S` は `FUN_140d174f0` の返値である。艦隊側の基礎値、所属艦の寄与、modifier をまとめた非負の水上探知集計値で、単純な艦数ではない。

### 1.2 実装式

```text
raw = S × A + (u - 0.5) × B

upper_capped = min(1, raw)
route_divisor = max(1, R × K)
fraction = upper_capped / route_divisor

Ncombat = max(1, round_away_from_zero(C × fraction))
```

MOD値を代入すると次の通りである。

```text
raw = S × 0.10 + [-0.075, +0.075)
route_divisor = max(1, R × 0.01)
```

重要な点は次の四つである。

1. `CONVOY_ATTACK_BASE_FACTOR = 0.15` は `S` へ掛からない。中央0の乱数幅を作り、結果へ最大で約±7.5ポイントを加える。
2. `S` へ掛かる主係数は `COMBAT_DETECTED_CONVOYS_FROM_SURFACE_DETECTION_STAT = 0.10` である。
3. `raw` には上限1があるが、コード上の下限0 clampはない。負値になっても、最後の `max(1, ...)` により最低1隻になる。
4. 隻数化は単純な切り捨てではない。`FUN_14243b2f0 @ 0x14243b2f0` が正負とも絶対値方向の四捨五入を行い、その後に最低1隻を保証する。

`upper_capped ≤ 1` かつ `route_divisor ≥ 1` なので、正の範囲では参加隻数が航路の輸送船総数を超えない。

### 1.3 数値例

`R ≤ 100` なら `route_divisor = 1` である。

| 実効水上探知 `S` | 参加割合の乱数範囲 | `C = 20` の参加隻数 |
|---:|---:|---:|
| 0 | -7.5%～7.5%未満 | 1隻 |
| 1 | 2.5%～17.5%未満 | 1～3隻 |
| 5 | 42.5%～57.5%未満 | 9～11隻 |
| 10 | 92.5%～100% | 19～20隻 |
| 10.75以上 | 常に100% | 20隻 |

したがって、このMODでは参戦規模を直接押し上げるのは実効水上探知 `S` である。defineのコメントだけを読んで `CONVOY_ATTACK_BASE_FACTOR` を「固定15%の基礎迎撃率」と解釈すると、実バイナリの式と一致しない。

## 2. 戦闘終了時の「損傷蓄積による追加沈没」

`FUN_141bc2f60 @ 0x141bc2f60` は、海戦終了処理で損傷した輸送船を追加で沈没させる専用処理である。`FUN_14154c6e0` は戦闘の両陣営へこの処理を実行してから、戦闘終了・解放処理へ進む。

### 2.1 抽選式

対象グループ内の有効な輸送船について、現在耐久割合を平均する。

```text
ri = 輸送船iの current_strength / max_strength
n  = 集計対象の輸送船数

average_strength = Σri / n
average_missing  = 1 - average_strength

Pspill = average_missing × CONVOY_SINKING_SPILLOVER
       = average_missing × 0.50
```

乱数が `Pspill` 未満なら追加沈没処理へ進む。無傷なら `average_missing = 0` なので抽選自体がない。

### 2.2 このMODでは追加沈没は最大1隻

一般形では、抽選通過後の沈没試行数は次である。

```text
Ksink = max(1, floor(Pspill))
```

ここで `Pspill` は割合表記である。このMODは係数が0.50で、`average_missing ≤ 1` なので、`Pspill ≤ 0.50`。したがって抽選に通っても、この処理が沈めるのは最大1隻である。

例として、同じグループに耐久100%、80%、60%、40%の輸送船がいる場合は次になる。

```text
average_strength = 70%
average_missing  = 30%
Pspill           = 30% × 0.50 = 15%
```

よって戦闘終了時に15%で、損傷済み輸送船をさらに1隻沈める。

### 2.3 対象選択

- 平均には無傷の輸送船も含まれる。
- 内部状態 `0x10` と `0x20` のメンバーは平均から除外される。
- 抽選通過後はグループのメンバー配列を末尾から走査し、耐久100%未満で、正式な輸送船撃沈処理に成功した最初の1隻を沈める。
- 被害船を改めてランダム選択する処理ではない。実質的な対象はメンバー配列順に依存する。

追加沈没は単なる戦闘結果画面上の補正ではない。`FUN_1418d4ff0` から通常の撃沈処理 `FUN_1418d47e0` へ入り、既存ログで確認済みの撃沈数・航路効率低下処理へ接続する。

## 3. `AGGRESSION_CONVOY_STRENGTH_FACTOR` の実オペランド

defineは次である。

```lua
AGGRESSION_CONVOY_STRENGTH_FACTOR = 0.3
```

名称とコメントは「輸送船のstrengthを攻撃性計算で割り引く」と読めるが、`FUN_1418e8a70 @ 0x1418e8a70` が実際に係数を掛ける値は、輸送船装備の `max_organisation` である。取得関数 `FUN_141a3e7b0` は装備定義オブジェクトの `+0x2f8` を返し、このoffsetは別経路でも最大Orgとして確認できる。

このMODの輸送船は `max_organisation = 30` なので、輸送船1隻が該当する戦闘陣営集計欄へ加える値は次になる。

```text
30 × 0.3 = 9
```

これは輸送船の `max_strength = 60` を18へ下げる処理ではなく、砲撃・雷撃ダメージを0.3倍にする処理でもない。海戦参加メンバーを陣営別に集計する `FUN_1415ab1e0` の評価用サマリーでのみ適用される。

下流の交戦継続・撤退判定で、この集計欄が最終的にどの閾値と比較されるかまでは今回確定していない。そのため「輸送船が何隻なら艦隊が撤退する」といった数値換算は行わない。

## 4. 上記処理を読むための輸送船実データ

`common/units/equipment/convoys.txt` の基礎値は次である。

| 項目 | MOD値 | 今回の意味 |
|---|---:|---|
| `max_strength` | 60 | 追加沈没式の各船の最大耐久 |
| `max_organisation` | 30 | 攻撃性集計で0.3倍される実値 |
| `naval_speed` | 12 | 輸送船の基礎速度 |
| `surface_visibility` | 14 | 輸送船側の水上視認性 |
| `reliability` | 0.8 | 艦船共通の信頼性入力 |
| `anti_air_attack` | 0.1 | 小さい対空値 |
| `surface_detection` | 0 | ファイル上も「convoyでは未使用」 |
| `sub_detection` | 0 | ファイル上も「convoyでは未使用」 |
| `torpedo_attack` | 0 | 攻撃力なし |
| `lg_attack` | 0 | 攻撃力なし |
| `offensive_weapons` | no | 攻勢兵装を持たない |
| `build_cost_ic` | 40 | 生産コスト |

輸送船自身の `surface_detection = 0` は、§1の `S` を0にする意味ではない。§1の `S` は輸送船を捕捉する襲撃任務部隊側の集計値である。

## 5. `convoy_retreat_speed` のMOD内供給源

MOD内で確認できる値は次の通りである。

| 供給源 | 値 |
|---|---:|
| screen doctrine本体 | +0.10 |
| `sea_scabs` reward | +0.10 |
| `evasive_convoy_maneuvers` reward | +0.50 |
| `blockade_runner` trait | +0.05 |
| 海軍 maneuvering skill 1～10 | +0.16～+0.25 |

バイナリ側では `convoy_retreat_speed` と `MODIFIER_CONVOY_RETREAT_SPEED` の登録を確認した。ただし、汎用modifier基盤を経由した後の「撤退進捗1tickあたりの時間」への変換式は、今回の追跡では確定できていない。よって、これらを単純加算して「撤退時間が何%短縮される」とは断定しない。

## 6. バイナリ根拠

| 内容 | 関数・アドレス |
|---|---|
| 通常輸送船海戦の作成と参戦隻数 | `FUN_140f6ec10 @ 0x140f6ec10` |
| 実効水上探知集計 | `FUN_140d174f0 @ 0x140d174f0` |
| `COMBAT_DETECTED...` のロード | `FUN_14091a500 @ 0x14091a500` → `DAT_14324bc50` |
| `CONVOY_ATTACK_BASE_FACTOR` のロード | `FUN_14091c3f0 @ 0x14091c3f0` → `DAT_143247198` |
| `CONVOY_ROUTE_SIZE...` のロード | `FUN_14091dcb0 @ 0x14091dcb0` → `DAT_14324bcf0` |
| 隻数の四捨五入 | `FUN_14243b2f0 @ 0x14243b2f0` |
| 終戦時の損傷蓄積抽選 | `FUN_141bc2f60 @ 0x141bc2f60` |
| 両陣営への終戦時適用 | `FUN_1415a97f0` / `FUN_14154c6e0` |
| 追加沈没から正式撃沈処理への接続 | `FUN_1418d4ff0` → `FUN_1418d47e0` |
| 輸送船の陣営評価集計 | `FUN_1418e8a70 @ 0x1418e8a70` |
| 最大Org取得 | `FUN_141a3e7b0 @ 0x141a3e7b0` |
| 両陣営メンバー集計 | `FUN_1415ab1e0 @ 0x1415ab1e0` |

## 7. 未確定として残す点

- `convoy_retreat_speed` が撤退時間へ変換される最終式と、複数供給源の実効stack規則
- 追加沈没平均から除外される内部状態 `0x10` / `0x20` の正式なenum名
- 攻撃性サマリー内の `max_organisation × 0.3` が最終撤退閾値へ入る下流式

この三点は、根拠が揃うまで数値効果を断定しない。
