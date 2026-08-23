# HOI4 空戦の撃墜計算と乱数処理

> **innovative balance v13.00 対応版**  
> 係数は Workshop ID `820039020` の `common/defines/00_defines.lua` を基準にしている。実行ファイル側の処理構造・関数アドレスは元のGhidra解析結果、数値はMOD値である。

## 結論

`hoi4.exe` の通常空戦における撃墜数計算本体は、Ghidra 上の
`FUN_140f4a790`（`0x140f4a790`、ソース位置表示は `airmission.cpp`）です。

処理は次の順です。

1. 攻撃側の Air Attack に、参加機数と対象への配分係数を掛ける。
2. `COMBAT_DAMAGE_STATS_MULTILPIER` を掛ける。
3. 速度優位による加算と、敏捷性劣位による減算を行う。
4. 防御側の Air Defence で割り、`COMBAT_DAMAGE_SCALE` を掛けて期待撃墜数 `E` を作る。
5. `floor(E)` は確定撃墜とし、小数部分だけ乱数で追加の1機を判定する。
6. 実際に交戦可能な防御側機数で上限をかける。

したがって、撃墜数そのものを一発の乱数で引く方式ではなく、**固定小数点で計算した期待値を確率的に丸める方式**です。

## 記号

- `A`：攻撃側の最終 Air Attack
- `D`：防御側の最終 Air Defence
- `GA`, `GD`：攻撃側・防御側の最終 Agility
- `SA`, `SD`：攻撃側・防御側の最終 Speed
- `n`：この対象に割り当てられた攻撃機数
- `F`：呼び出し元が対象ごとに作る配分係数。距離・対象地域に関係する分岐を含む
- `C = n × A × F`：撃墜計算本体へ渡される攻撃寄与値
- `Kmax`：この攻撃で交戦・撃墜可能な防御側機数の上限

ゲーム内部では実数 `1.0` を整数 `100000` とする固定小数点が使われています。以下は通常の実数表記へ直した式です。

## 撃墜計算式

### 1. 基礎ダメージ

```text
B = C × COMBAT_DAMAGE_STATS_MULTILPIER
```

MODの `common/defines/00_defines.lua` では、

```text
COMBAT_DAMAGE_STATS_MULTILPIER = 0.2
```

なので `B = 0.2 × C` です。define 名の `MULTILPIER` は実ファイル中でもこの綴りです。

### 2. 敏捷性による減算

攻撃側の Agility が防御側以上なら減算はありません。攻撃側が劣る場合は、

```text
agilityRatio = min(GD / GA, BIGGEST_AGILITY_FACTOR_DIFF)
agilityPenalty = (agilityRatio - 1)
               × COMBAT_BETTER_AGILITY_DAMAGE_REDUCTION
               × B
```

MOD値を入れると、

```text
agilityPenalty = (min(GD / GA, 2.0) - 1) × 0.9 × B
```

このロジックは `FUN_140f4a790` にインライン化されています。同じ計算を行う独立ヘルパーは `FUN_140f3bb90`（`0x140f3bb90`）です。

### 3. 速度による加算

攻撃側の Speed が防御側以下なら加算はありません。攻撃側が速い場合、

```text
speedRatio = min(SA / SD, BIGGEST_SPEED_FACTOR_DIFF)

speedBonus = [ (SA / 100) × TOP_SPEED_DAMAGE_BONUS_FACTOR
             + (speedRatio - 1) × COMBAT_BETTER_SPEED_DAMAGE_INCREASE ]
           × B
```

MOD値を入れると、

```text
speedBonus = [ (SA / 100) × 0.000
             + (min(SA / SD, 3.5) - 1) × 1.8 ] × B
           = (min(SA / SD, 3.5) - 1) × 1.8 × B
```

この計算は `FUN_140f3f240`（`0x140f3f240`）です。

### 4. 期待撃墜数

```text
effectiveDamage = B + speedBonus - agilityPenalty

Eraw = 0.01 × damageScale × effectiveDamage / D
E    = max(Eraw, 0.001)
```

`damageScale` のMOD値は通常空戦で `COMBAT_DAMAGE_SCALE = 1.00`、**攻撃側と対象側の両方が空母戦闘用エントリである場合**には `COMBAT_DAMAGE_SCALE_CARRIER = 3.00` です。片側だけが基地航空隊ならこの組合せ条件は成立せず、通常値1.00を使います。

通常空戦なら、全体は次の形になります。

```text
E = max(
      0.001,
      0.01 × 0.2 × (n × A × F) / D
      × [1 + speedFactor - agilityFactor]
    )
```

ここで、

```text
speedFactor = 0                                      (SA <= SD)
            = (min(SA/SD,3.5)-1)×1.8                (SA > SD)

agilityFactor = 0                                   (GA >= GD)
              = (min(GD/GA,2.0)-1)×0.9              (GA < GD)
```

### 5. 乱数による確率的丸め

`E < Kmax` のとき、

```text
base = floor(E)
frac = E - base
R    = random_fixed()       // 0.00000 ～ 0.99999

kills = base + (R < frac ? 1 : 0)
```

`E >= Kmax` なら `kills = Kmax` です。

つまり `E = 2.37` なら、2機は確定で、残り37%の確率で1機増えます。期待値は2.37機のままです。内部比較は10万分率なので、確率の刻みは `1/100000` です。

## 交戦機数の上限

呼び出し元は `FUN_140f40320`（`0x140f40320`）です。ここでは概ね、

```text
engagedAttackers = min(attackerPlanes,
                       totalDefenderPlanes × COMBAT_MULTIPLANE_CAP)
```

としてから、各防御側ウイングの機数比で攻撃機と Air Attack を配分しています。MOD値 `COMBAT_MULTIPLANE_CAP = 1.0` のため、1機の防御機に同時に割り当てられる攻撃機は最大1機相当です。

対象別に渡される攻撃寄与値は、コード上では次の形です。

```text
Cj = engagedAttackers
   × (defenderPlanes_j / totalDefenderPlanes)
   × attackerAirAttack
   × Fj
```

`Fj` は対象地域・距離を参照して作られる係数ですが、今回の段階では既存のゲーム内名称までは確定していません。撃墜本体 `FUN_140f4a790` へ入った後の式は上記のとおり確定しています。

## 乱数生成器

撃墜数の小数部を丸める乱数呼び出しは、

- 呼び出し箇所：`0x140f4ab03`
- 呼び出し元：`FUN_140f4a790`
- Ghidra のソース位置表示：`airmission.cpp:0xacd`
- 呼び出される関数：`FUN_142187010`（`0x142187010`）
- 乱数関数側のソース位置表示：`clausewitzlib/random.cpp`

です。

`FUN_142187010` は、グローバル状態

- カウンタ：`DAT_143366310`
- シード／キー：`DAT_143366314`

を混合して値を生成し、カウンタを1増やした後、最終的に

```text
result = mixedValue & 0x7fffffff
result = result % 100000
```

を返します。戻り値の範囲は `0..99999` です。

関数内には、乱数禁止領域から呼ぶと Out Of Sync の原因になる、という Clausewitz 側のアサート文字列もあります。このため、これはOSの真性乱数ではなく、同期可能なゲームプレイ用の**決定論的PRNG**です。

同じ撃墜処理の後半には、撃墜数そのものとは別に、エース関連の選択・帰属処理と思われる50%判定でも同じ乱数関数が呼ばれています（`0x140f4ab51`、`0x140f4ac48`）。

## 確認できた stat の対応

`FUN_140f23c00`（`0x140f23c00`）が戦闘用 stat 配列を作っています。

```text
output[0] = final Air Attack
output[1] = final Air Defence
output[2] = final Agility
output[3] = Air Attack  / AIR_WING_MAX_STATS_ATTACK
output[4] = Air Defence / AIR_WING_MAX_STATS_DEFENCE
output[5] = Agility     / AIR_WING_MAX_STATS_AGILITY
```

撃墜本体は `output[1]` を防御値として、`output[5]` を敏捷性比の計算に使っています。同じ最大値で正規化された値同士を割るため、通常範囲では `GD/GA` と同値です。

## 使用される主な define のMOD値

今回の実行ファイルから define の参照先を追い、`innovative balance` の `00_defines.lua` の値と対応させました。

| define | MOD値 |
|---|---:|
| `AIR_WING_MAX_STATS_ATTACK` | 100 |
| `AIR_WING_MAX_STATS_DEFENCE` | 100 |
| `AIR_WING_MAX_STATS_AGILITY` | 100 |
| `AIR_WING_MAX_STATS_SPEED` | 1000 |
| `BIGGEST_AGILITY_FACTOR_DIFF` | 2.0 |
| `BIGGEST_SPEED_FACTOR_DIFF` | 3.5 |
| `TOP_SPEED_DAMAGE_BONUS_FACTOR` | 0.000 |
| `COMBAT_DAMAGE_STATS_MULTILPIER` | 0.2 |
| `COMBAT_BETTER_AGILITY_DAMAGE_REDUCTION` | 0.9 |
| `COMBAT_BETTER_SPEED_DAMAGE_INCREASE` | 1.8 |
| `COMBAT_MULTIPLANE_CAP` | 1.0 |
| `COMBAT_DAMAGE_SCALE` | 1.00 |
| `COMBAT_DAMAGE_SCALE_CARRIER` | 3.00 |

`AIR_COMBAT_FINAL_DAMAGE_SCALE`、`AIR_COMBAT_FINAL_DAMAGE_PLANES`、`AIR_COMBAT_FINAL_DAMAGE_PLANES_FACTOR` という別の define も登録されていますが、今回追跡した通常空戦の撃墜本体からは直接参照されていません。少なくともこの経路へ混ぜて式を作るべきではありません。

## 空母戦闘での適用範囲

`COMBAT_DAMAGE_SCALE_CARRIER` は「空母機なら常に撃墜力が上がる」係数でも、「艦船へ向かう爆撃機を迎撃しやすくする確率」でもありません。`FUN_140f40320` が攻撃側・対象側の航空戦エントリを比較し、双方が空母戦闘フラグを持つ組合せだけについて、`FUN_140f4a790` 内の通常scaleを空母scaleへ差し替えます。

```text
carrierCombatPair = attacker.carrierCombat
                 && defender.carrierCombat

damageScale = carrierCombatPair
            ? COMBAT_DAMAGE_SCALE_CARRIER
            : COMBAT_DAMAGE_SCALE
```

現在のバニラ値は5.00、innovative balance v13.00は3.00です。通常空戦のMOD値1.00と比べると、空母戦闘用の組合せでは各射撃方向の期待撃墜数が3倍になります。相互空戦では両陣営が順に攻撃側になるため、全体としては双方の損害を増やす対称的な致死率補正です。

これとは別に、艦艇攻撃直前には防御空母の艦上戦闘機が来襲機だけを撃つ一方向CAP処理 `FUN_1418daad0` があります。このCAPは本資料の `FUN_140f4a790` を使わず、次の別係数を使います。

```text
CARRIER_PERCENTAGE_DEFEND              = 0.35
CARRIER_COMBAT_DAMAGE_STATS_MULTIPLIER = 0.30  # MOD
```

したがって、`COMBAT_DAMAGE_SCALE_CARRIER = 3.00` と `CARRIER_COMBAT_DAMAGE_STATS_MULTIPLIER = 0.30` は同じ効果ではありません。前者は空母航空隊同士の通常空戦コア、後者は艦艇到達直前の一方向CAPです。

## 確定度

- **確定**：撃墜計算本体、Air Attack／Defence／Agility の対応、速度・敏捷性補正、期待値の確率的丸め、乱数関数と状態アドレス、空母scaleの双方フラグ条件。
- **強い推定**：`FUN_140f40320` が渡す対象別係数 `Fj` の意味。地域距離を参照していることは確認済みですが、ゲーム内での正式名称は未確定です。
- **未調査**：命中対象の選択順、任務効率・探知・迎撃参加率が `n` と `Fj` に到達するまでの上流全体、エース帰属処理の詳細。

## 主なアドレス一覧

| アドレス | 内容 |
|---|---|
| `0x140f40320` | 攻撃機数・攻撃値を防御側ウイングへ配分 |
| `0x140f4a790` | 通常空戦の撃墜計算本体 |
| `0x140f4ab03` | 撃墜期待値の小数部を丸める乱数呼び出し |
| `0x140f3bb90` | 敏捷性劣位によるダメージ減算ヘルパー |
| `0x140f3f240` | 速度優位によるダメージ加算ヘルパー |
| `0x140f23c00` | Air Attack／Defence／Agility の戦闘用 stat 取得 |
| `0x142187010` | `0..99999` を返す決定論的PRNG |
| `0x143366310` | PRNG カウンタ |
| `0x143366314` | PRNG シード／キー |
