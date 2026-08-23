# HOI4 海軍クリティカルヒット完全解析

> **対象:** HOI4 1.17.5.2 系の `hoi4.exe` と、Workshop ID `820039020` の **innovative balance v13.00**。  
> **範囲:** 海戦中の軽砲・重砲・艦船魚雷、海戦内／戦略地域／港湾への海軍航空攻撃、クリティカル部位、関連modifier、修理解除、海軍事故。  
> 実行ファイルの処理はGhidraによる逆コンパイル、係数はバニラおよびMODのローカル定義を根拠とする。

## 結論

1. **艦砲、艦船魚雷、航空攻撃では、クリティカル率もクリティカル時のダメージも別の式を使う。**
2. **信頼性は砲撃と航空攻撃のクリティカル率を下げ、汎用クリティカルのダメージ倍率も下げる。艦船魚雷の基礎クリティカル率と魚雷専用倍率には信頼性が入らない。**
3. **装甲／貫徹比を参照するのは艦砲だけである。**
   - 艦船魚雷は装甲計算を専用値 `-1.0` で迂回する。
   - 航空攻撃は実際の装甲値と航空機側貫徹値を比較せず、`COMBAT_ARMOR_PIERCING_CRITICAL_BONUS` を常時使う。
4. **クリティカル成立後、さらに「部位損傷を付けるか」の独立抽選がある。**
   - 水上砲・艦船魚雷: `CHANCE_TO_DAMAGE_PART_ON_CRITICAL_HIT / reliability`
   - 航空攻撃: `CHANCE_TO_DAMAGE_PART_ON_CRITICAL_HIT_FROM_AIR / reliability`
   - 全通常艦種の艦種係数は1.0なので省略した形。低信頼性ほど部位損傷まで起きやすい。
5. **部位損傷に成功した場合、汎用クリティカル倍率は重ならない。** 部位固有の倍率・固定ダメージと持続デバフを適用する。部位抽選に失敗した場合、または損傷可能な部位が残っていない場合だけ、兵装別の汎用クリティカル倍率を使う。
6. **innovative balance v13.00 はバニラと比べ、低頻度・高威力の砲クリティカル、極端に高頻度・高威力の魚雷クリティカル、そして高い部位損傷率へ変更している。**

## 1. 全体の処理順

海戦攻撃は概略として次の順に処理される。

```text
攻撃実行
  ↓
命中判定
  ↓ 命中時のみ
基礎STR・ORGダメージと通常ダメージ乱数を計算
  ↓
兵装別クリティカル率を計算
  ↓
クリティカル乱数
  ├─ 失敗 → 通常ダメージ
  └─ 成功
       ↓
     部位損傷ゲート乱数
       ├─ 成功かつ候補あり
       │    ↓
       │  重み付きで部位を1種類選ぶ
       │    ↓
       │  部位固有の当該一撃ダメージ + 持続デバフ
       │  ※汎用クリティカル倍率は適用しない
       └─ 失敗または候補なし
            ↓
          兵装別の汎用クリティカル倍率
```

したがって、以下の式で示すクリティカル率は、原則として**命中した攻撃1回を条件とする確率**である。発射1回から見た最終確率は、おおむね `命中率 × クリティカル率` になる。

## 2. 記号

| 記号 | 意味 |
|---|---|
| `R` | 目標艦の信頼性。80%なら `0.80` |
| `A` | 艦砲の `weapon piercing / target armor` |
| `K(A)` | 貫徹比から選ぶ艦砲用クリティカル係数 |
| `S` | 攻撃側の `naval_critical_score_chance_factor` と対国補正の合計 |
| `Q` | 目標側の `critical_receive_chance`。防御ボーナスは通常負値 |
| `T` | 目標装備側の `naval_torpedo_enemy_critical_chance_factor` |
| `N` | 1回の航空攻撃グループで実際に攻撃ダメージへ参加した機数 |

ゲーム内部は10万を1.0とする固定小数で計算する。以下は読みやすい実数表記へ直している。確率が0未満なら0、1を超えれば乱数上は実質100%となる。

## 3. 軽砲・重砲のクリティカル

### 3.1 発生率

軽砲と重砲は同じクリティカル式を使う。

```text
Pcrit_gun = clamp01(
    (1 - R)
    × COMBAT_BASE_CRITICAL_CHANCE
    × K(A)
    × (1 + S + Q)
)
```

`S` には通常の「クリティカルヒットの確率」と、存在する場合は攻撃国対防御国の対国補正が加算される。`Q` は同じ括弧へ加算されるため、たとえば `critical_receive_chance = -0.30` は、他の補正がなければ最終クリティカル率を0.70倍にする。

### 3.2 装甲／貫徹の段階表

`FUN_1418cecb0` は線形補間をしていない。`A` が到達した最初の閾値に対応する値を、そのまま返す段階関数である。

| 艦砲の貫徹比 `A` | バニラ `K(A)` | MOD `K(A)` |
|---:|---:|---:|
| `A >= 2.00` | 2.00 | 1.00 |
| `1.00 <= A < 2.00` | 1.00 | 1.00 |
| `0.75 <= A < 1.00` | 0.75 | 0.75 |
| `0.50 <= A < 0.75` | 0.50 | 0.50 |
| `0.10 <= A < 0.50` | 0.10 | 0.10 |
| `A < 0.10` | 0.00 | 0.00 |

装甲0は最大段階として扱われる。バニラでは2倍だが、MODは最上段を1.0へ変更しているので、2倍オーバーマッチによる追加クリティカルボーナスはない。

### 3.3 部位が付かなかった場合のダメージ倍率

```text
Mfallback_gun = 1 + (1 - R) × COMBAT_CRITICAL_DAMAGE_MULT
```

これはdefine値そのものを掛けるのではなく、**1を足したうえで信頼性により縮小した倍率**である。

| 信頼性 | バニラ (`MULT=5`) | MOD (`MULT=15`) |
|---:|---:|---:|
| 70% | 2.50倍 | 5.50倍 |
| 80% | 2.00倍 | 4.00倍 |
| 90% | 1.50倍 | 2.50倍 |
| 100% | 1.00倍。ただし発生率も0 | 1.00倍。ただし発生率も0 |

### 3.4 MODでの例

信頼性80%、クリティカル関係modifierなしの場合:

| 貫徹比 | クリティカル率／命中 | 部位なし時の倍率 |
|---:|---:|---:|
| `A >= 1.00` | 0.20% | 4.00倍 |
| `0.75 <= A < 1.00` | 0.15% | 4.00倍 |
| `0.50 <= A < 0.75` | 0.10% | 4.00倍 |
| `0.10 <= A < 0.50` | 0.02% | 4.00倍 |
| `A < 0.10` | 0% | 発生しない |

MODでは砲の基礎率を5%から1%へ落とす一方、汎用ダメージ係数を5から15へ上げている。つまり砲は「出にくいが、出た場合の上振れが大きい」設計である。

## 4. 艦船魚雷のクリティカル

### 4.1 発生率

艦船の `torpedo_attack` は信頼性と装甲／貫徹表を使わない。

```text
Pcrit_torpedo = clamp01(
    COMBAT_TORPEDO_CRITICAL_CHANCE
    × (1 + T)
    × (1 + S + Q)
)
```

- `T` は目標艦装備の `naval_torpedo_enemy_critical_chance_factor`。
- `S` と `Q` は砲撃と共通の最終補正。
- `R` と `K(A)` は入らない。

### 4.2 部位が付かなかった場合のダメージ倍率

```text
Mfallback_torpedo = COMBAT_TORPEDO_CRITICAL_DAMAGE_MULT
```

これは固定倍率であり、砲と違って `1 + ...` ではなく、信頼性でも縮小されない。

| 環境 | 基礎クリティカル率 | 部位なし時の倍率 |
|---|---:|---:|
| バニラ | 10% | 2倍 |
| innovative balance v13.00 | 60% | 15倍 |

MODでは、modifierなしなら**命中魚雷の60%がクリティカル判定に成功**する。ただし後述の部位損傷に成功すると15倍は使われず、選ばれた部位の倍率へ置き換わる。

### 4.3 「魚雷の水上艦貫徹」との関係

`naval_torpedo_screen_penetration_factor`、表示名「魚雷の水上艦貫徹」は標的選択時に直衛を抜ける確率の補正であり、上の式へは入らない。装甲貫徹、1発のダメージ、命中率、クリティカル率を直接増やす効果ではない。

## 5. 艦上攻撃機・基地航空隊の対艦攻撃クリティカル

### 5.1 どの航空攻撃が同じ経路か

次は最終的に同じ航空対艦ダメージ関数 `FUN_1418da3c0` を使う。

- 海戦内で空母から発進する艦上攻撃機
- 戦略地域から艦隊を爆撃する基地航空隊
- 港湾攻撃側から渡される海軍航空攻撃

航空機が雷撃機に見えても、ここでは艦船の `torpedo_attack` ではなく航空側 `naval_attack` として処理する。そのため、`COMBAT_TORPEDO_CRITICAL_CHANCE` と `COMBAT_TORPEDO_CRITICAL_DAMAGE_MULT` は参照しない。

航空ダメージ束へ渡されるNaval Attackは設計画面の表示値そのままではなく、Naval Attack getterで0.1倍された内部値である。

```text
airAttackInternal = displayedNavalAttack × 0.1 × statModifiers
```

ただし、この攻撃値の大きさ自体は下記の航空クリティカル**発生率**へ入らない。発生したクリティカルが掛かる基礎ダメージ束の大きさへ影響する。

### 5.2 発生率

実行ファイルの式は次の形である。

```text
Bair = (1 - min(R, 1))
       × COMBAT_BASE_CRITICAL_CHANCE
       × (1 + COMBAT_ARMOR_PIERCING_CRITICAL_BONUS)

Pair_raw = Bair × max(0, 1 - Q)

Pcrit_air =
    0                                      if Bair == 0
    max(0.001, Pair_raw)                   if Bair > 0
```

重要点:

- 名前に `ARMOR_PIERCING` とあるが、航空経路は目標装甲も航空機の実貫徹値も比較しない。このbonusを常時掛ける。
- 艦砲用 `NAVY_PIERCING_THRESHOLD_CRITICAL_VALUES` は使わない。
- 攻撃側の `naval_critical_score_chance_factor` と対国クリティカル補正も、この航空経路では読まない。
- 正の基礎率がある限り、目標補正後の値が低すぎても0.1%へ引き上げる。信頼性100%では `Bair=0` なので床値も発生しない。

### 5.3 `critical_receive_chance` の符号に関する実装上の不整合

艦砲／艦船魚雷は `× (1 + ... + Q)` だが、航空経路の機械語は `× (1 - Q)` になっている。同じゲームデータでは、防御的な値を `critical_receive_chance = -0.1` のような負値で定義している。

したがって、実行ファイルを字義どおり読むと:

```text
Q = -0.30

surface factor = 1 + Q = 0.70   # 砲・艦船魚雷は30%減
air factor     = 1 - Q = 1.30   # 航空攻撃は30%増
```

これは少なくとも処理間で符号が一致しておらず、**`FUN_1418da3c0` が受け取る値をスクリプト値と同じ符号の `Q` と解釈する限り、航空攻撃側では防御modifierが逆効果になる実装**である。ゲーム内日本語表示は「クリティカルヒットを受ける確率」、説明文は「海戦でクリティカルヒットを受ける確率を低下」であり、データも防御効果を負値で定義している。

#### 再検証で確認したこと

この結論はGhidraの高水準擬似コードの `-` 記号だけに依存していない。対象exeの航空経路には減算命令があり、水上経路の加算と実際に異なる。また同じMODではdamage control技術等が `critical_receive_chance = -0.1` と定義されているため、単なる資料上の正負取り違えでもない。

ただし、ここから直ちに「Paradoxのバグである」と100%断定するには、実行時にgetterが返す値まで観測するか、十分な標本数のA/B試験が必要である。残る代替可能性は次の通り。

- 航空経路へ渡る直前にだけgetterまたは集計器が符号を反転している。
- `critical_receive_chance` と同定した数値IDが、航空経路でのみ逆符号の意味を持つ。
- 別の上流補正や上限処理が、実ゲーム上の最終率を打ち消している。

現在の静的解析ではこれらを積極的に示す証拠はなく、**航空経路の符号ミスが最有力**である。一方、「表示上の意図と機械語が食い違う」という事実と、「実ゲームで必ず逆効果になる」という最終実証は分けて扱う。

#### ゲーム内ストレステストの読み方

damage controlの値を極端な負値へ変更した試験で、航空攻撃によるクリティカルが実際に記録されたことは、少なくとも「負値を大きくすれば航空クリティカルが完全に消える」という素朴な期待とは合わない。ただし戦果UIの「航空機: 9」は通常、9回の独立試行ではなく航空機による合計ダメージ表示である。また「受けたクリティカルヒット数」は部位・イベントの集計であり、分母9の二項試験として `2/9` をそのまま率にできない。

符号をゲーム上で確定するには、他条件を固定して `Q=0` と大きな負値の2条件を多数回繰り返し、航空クリティカル件数を比較する必要がある。damage control 1だけを変更してその技術だけ取得する方法は、3段階分の加算や他効果を避け、どの変更が結果を生んだか追跡しやすくするためである。`-1000` のような極値もストレステストには使えるが、確率clamp、部位数上限、UI集計単位の影響により線形な1000倍差を期待すべきではない。

### 5.4 部位が付かなかった場合のダメージ倍率

```text
Mfallback_air = 1 + (1 - R)
                   × COMBAT_CRITICAL_DAMAGE_MULT
                   / N
```

航空攻撃は複数機分をまとめたダメージ束に対してクリティカルを1回抽選するため、汎用倍率の追加分を参加機数 `N` で割る。

MOD、信頼性80%なら:

```text
Mfallback_air = 1 + 3 / N
```

例として `N=1` なら4倍、`N=10` なら1.30倍、`N=20` なら1.15倍である。逆に弾薬庫誘爆などの部位が選ばれた場合、部位固有倍率はまとめられた航空ダメージ束そのものへ掛かる。

### 5.5 バニラとMODの基礎率

modifierなし、信頼性80%の例:

| 環境 | 計算 | クリティカル率／航空攻撃グループ |
|---|---|---:|
| バニラ | `(1-.8) × .05 × (1+1)` | 2% |
| MOD | `(1-.8) × .01 × (1+19)` | 4% |

MODの `COMBAT_ARMOR_PIERCING_CRITICAL_BONUS=19` により、基礎1%を20倍してから信頼性で縮小する。

## 6. クリティカル後の部位損傷ゲート

クリティカル乱数に成功しても、部位はまだ確定しない。目標が通常の `CShip` なら、次の確率でもう一度抽選する。

### 6.1 砲・艦船魚雷

```text
Ppart_surface = clamp01(
    CHANCE_TO_DAMAGE_PART_ON_CRITICAL_HIT
    × critical_part_damage_chance_mult
    / R
)
```

### 6.2 航空攻撃

```text
Ppart_air = clamp01(
    CHANCE_TO_DAMAGE_PART_ON_CRITICAL_HIT_FROM_AIR
    × critical_part_damage_chance_mult
    / R
)
```

現在の通常艦種はすべて `critical_part_damage_chance_mult = 1`。信頼性0以下では除算せず部位選択へ直行するため、実質100%である。

| 環境 | 水上砲・艦船魚雷の基礎 | 航空攻撃の基礎 | 信頼性80%時 |
|---|---:|---:|---:|
| バニラ | 10% | 10% | 両方12.5% |
| MOD | 50% | 25% | 水上62.5%、航空31.25% |

このMODでは、特に砲・艦船魚雷クリティカルから部位損傷へ進む率がバニラの5倍になっている。

### 6.3 輸送船

輸送船オブジェクトはこの `CShip` 部位処理へ入らない。輸送船にクリティカルが成立した場合は、永続的な砲塔・推進器などを付けず、兵装別の汎用クリティカル倍率へフォールバックする。

## 7. 損傷部位の候補と重み付き抽選

部位ゲートに成功すると、ゲームは次を候補へ集める。

1. 艦種そのものが持つ基礎部位
2. 搭載モジュールが追加した部位
3. まだ最大損傷回数へ到達していない部位だけを残す
4. `chance` を重みとして1種類を抽選する

条件付き選択確率は:

```text
P(select part i | part gate passed)
    = weight_i / sum(weight of all currently eligible part types)
```

`chance` 省略時の重みは1.0。複数の同種砲塔を積むと、その部位を損傷できる**回数上限**は増えるが、同一部位タイプが候補リストへ何重にも入り重みが単純倍増する処理ではない。

### 7.1 艦種の基礎部位

| 艦種 | 基礎部位 |
|---|---|
| 戦艦・巡洋戦艦・重巡・軽巡・駆逐艦 | 弾薬庫、推進器、火災、舵 |
| 空母 | 弾薬庫、推進器、火災 |
| 潜水艦 | 弾薬庫、推進器、火災、舵、バラストタンク |

### 7.2 モジュールが加える部位

| モジュール系統 | 追加部位 |
|---|---|
| 主に軽砲系 | `damaged_light_guns` |
| 副砲 | `damaged_secondaries` |
| 重砲系 | `damaged_heavy_guns` |
| 水上・潜水艦魚雷発射管 | `damaged_torpedoes` |

## 8. 全クリティカル部位の効果

部位選択時の当該一撃は次の順序で変形される。

```text
newDamage = oldDamage × part_damage_multiplier + part_flat_damage
```

部位が選ばれた場合、砲4倍、魚雷15倍などの汎用倍率はさらに掛からない。

| 部位 | 重み | 当該一撃のSTR | 当該一撃のORG | 持続効果 |
|---|---:|---:|---:|---|
| 重砲損傷 | 1.0 | 1.0倍 | 1.0倍 | 重砲攻撃 -50%／損傷instance |
| 軽砲損傷 | 1.0 | 1.0倍 | 1.0倍 | 軽砲攻撃 -50%／instance |
| 副砲損傷 | 1.0 | 1.0倍 | 1.0倍 | 軽砲攻撃 -25%／instance |
| 魚雷発射管損傷 | 1.0 | 1.25倍 | 1.25倍 | 魚雷攻撃 -50%／instance |
| 弾薬庫誘爆 | 0.1 | 10倍 | 10倍 | 重砲・軽砲・魚雷攻撃 -70% |
| 火災 | 1.0 | 1.0倍 | `2倍 + 5` | 最大ORG -50%、ORG回復 -80% |
| 推進器損傷 | 0.5 | 0.5倍 | 0.5倍 | 速度 -90% |
| 舵損傷 | 0.25 | 0.25倍 | 0.25倍 | 撤退確率 -90%、速度 -50% |
| バラストタンク使用不能 | 1.0 | 0.25倍 | 0.25倍 | 潜水艦被発見性 +66% |

`弾薬庫誘爆` の重みは0.1なので他部位より選ばれにくいが、一撃10倍と非常に大きい。一方、推進器・舵・バラストの「クリティカル」は当該一撃だけを見ると通常攻撃より小さくなる。その代わり、速度・撤退・被発見性に深刻な持続効果を残す。

### 8.1 同一部位の重複

- 砲・魚雷モジュール部位は、最初の1instanceに加え、同系モジュールが1個増えるごとに損傷可能instanceが1増える。
- `stat_penalties` はinstance数に対して加算してから係数化する。例: 重砲損傷2instanceなら `1 + (-0.5 × 2) = 0` となり、重砲攻撃は0まで落ちる。
- 係数は負にならないよう0で下限処理される。
- **別の部位タイプ**が同じstatを下げる場合は、部位ごとに順番に乗算される。重砲損傷1instanceと弾薬庫誘爆が同時にあるなら、重砲攻撃は `0.5 × 0.3 = 0.15` になる。二部位の値を合算して0へclampする処理ではない。

## 9. `naval_critical_effect_factor` が実際に変えるもの

このmodifierは、クリティカルの**発生率**や、成立した一撃のSTR／ORG倍率を変えない。`critical_parts` 内の `modifier = { ... }` に記述された持続効果を次の形で拡縮する。

```text
persistent_modifier_effect
    = part_modifier_value
    × damaged_instances
    × (1 + naval_critical_effect_factor)
```

たとえば `naval_critical_effect_factor = -0.50` なら:

- 火災の最大ORG -50% → -25%
- 火災のORG回復 -80% → -40%
- 推進器損傷の速度 -90% → -45%

この倍率は0以上へclampされない。

```text
effect scale = 1 + naval_critical_effect_factor

合計 -1.00 → 0.00  対象効果が消える
合計 -1.10 → -0.10  ペナルティがボーナスへ反転する
```

`FUN_140bca3a0` の `0x140bca76f` で固定小数点の1.0を加えた後、`0x140bca7dd`～`0x140bca7f5` で部位modifierへ乗算するが、下限clampは存在しない。例えば合計-1.10なら、推進器損傷の速度-90%は速度+9%、火災の最大Org-50%は最大Org+5%の寄与へ反転する。

ただし次は縮小しない。

- `stat_penalties` に入る重砲・軽砲・魚雷攻撃低下、潜水艦被発見性
- 部位選択時の10倍、1.25倍、0.5倍などの即時ダメージ
- 火災の固定ORGダメージ `+5`

damage control技術が `critical_receive_chance` と `naval_critical_effect_factor` の両方を持つのは、前者が発生率、後者が付いた後の一部持続効果を別々に軽減するためである。

MODではDamage Control三段階の合計-0.30、`crisis_magician` の-0.50、最大艦経験値bonusの-0.30を重ねるだけで合計-1.10へ到達する。最大経験を使わなくても、満額の `naval_repair_support = -0.25` を加えれば合計-1.05である。後者の組み合わせはバニラにも存在するため、MOD固有の部位定義ではなく、対象exe側の下限clamp漏れである可能性が高い。

候補weight、汎用倍率置換、符号反転の数値例と試験方法は [hoi4_miscellaneous_reverse_engineering_notes.md](./hoi4_miscellaneous_reverse_engineering_notes.md) の「クリティカルの『部位損傷』再検証」節にまとめた。

## 10. 部位はいつ消えるか

クリティカル部位は海戦終了だけでは消えず、艦の状態へ保存される。修理処理 `FUN_140e75810` は耐久が99.999%以上へ達した時点で `FUN_140bc66b0` を呼び、保持している全クリティカル部位instanceを消去して艦性能を再計算する。

つまり実用上は:

```text
戦闘離脱
  ↓
港へ帰還して修理
  ↓
耐久が全回復
  ↓
クリティカル部位を一括解除
```

途中まで修理しただけでは、確認した経路上は全解除されない。

## 11. バニラとMODの主要係数比較

| define | バニラ | innovative balance v13.00 | 主な用途 |
|---|---:|---:|---|
| `COMBAT_BASE_CRITICAL_CHANCE` | 0.05 | 0.01 | 砲・航空の基礎率 |
| `COMBAT_CRITICAL_DAMAGE_MULT` | 5 | 15 | 砲・航空の汎用倍率追加分 |
| `COMBAT_ARMOR_PIERCING_CRITICAL_BONUS` | 1 | 19 | 現exeでは航空クリティカルに直接使用 |
| `COMBAT_TORPEDO_CRITICAL_CHANCE` | 0.10 | 0.60 | 艦船魚雷の基礎率 |
| `COMBAT_TORPEDO_CRITICAL_DAMAGE_MULT` | 2 | 15 | 艦船魚雷の汎用倍率 |
| `CHANCE_TO_DAMAGE_PART_ON_CRITICAL_HIT` | 0.10 | 0.50 | 砲・艦船魚雷の部位ゲート |
| `CHANCE_TO_DAMAGE_PART_ON_CRITICAL_HIT_FROM_AIR` | 0.10 | 0.25 | 航空攻撃の部位ゲート |
| 貫徹比2.0以上の砲クリティカル表 | 2.0 | 1.0 | 艦砲のオーバーマッチ段階 |

`critical_parts/00_critical_parts.txt` の部位効果そのものは、MODとバニラで同一である。大きく変えられているのは、発生率、汎用倍率、部位へ進む確率である。

## 12. MODで信頼性80%の比較

modifierなし、艦砲は貫徹比1.0以上、航空は `N=1` の単機相当として比較する。

| 攻撃 | クリティカル率／命中または攻撃束 | 部位ゲート／クリティカル | 部位なし時倍率 |
|---|---:|---:|---:|
| 軽砲・重砲 | 0.20% | 62.5% | 4倍 |
| 艦船魚雷 | 60% | 62.5% | 15倍 |
| 海軍航空攻撃 | 4% | 31.25% | `1 + 3/N` |

この表から分かる実戦上の要点:

- 砲クリティカルはかなり稀だが、発生すると部位損傷へ進みやすい。
- 魚雷はクリティカルそのものが非常に多い。さらに部位候補が残っている間は、その多くが弾薬庫・火災・推進器・舵・兵装損傷へ変換される。
- 「魚雷クリティカル=常に15倍」ではない。信頼性80%なら、候補がある限りクリティカルの62.5%はまず部位固有処理を試す。
- 航空の汎用倍率は機数で薄まるが、部位誘爆を引くと集約ダメージ全体に部位倍率が掛かる。

## 13. 関連modifier一覧

| modifier | 実装上の役割 |
|---|---|
| `naval_critical_score_chance_factor` | 攻撃側。砲・艦船魚雷の最終クリティカル率の括弧へ加算。航空経路は未参照 |
| `naval_critical_score_chance_factor_against_a_country` | 特定国への攻撃補正。砲・艦船魚雷の同じ括弧へ加算 |
| `critical_receive_chance` | 日本語表示「クリティカルヒットを受ける確率」。目標側。砲・艦船魚雷では負値が受ける率を低下。航空経路は減算しており符号不整合あり |
| `naval_torpedo_enemy_critical_chance_factor` | 目標装備側。艦船魚雷の専用基礎率を相対補正 |
| `naval_critical_effect_factor` | 付与済み部位の `modifier` ブロックにある持続効果を拡縮 |
| `reliability` | 砲・航空の発生率と汎用倍率を低下。全攻撃の部位損傷ゲートを逆数で低下 |

MOD内の代表例:

- `brave_commanders_spirit`: `naval_critical_score_chance_factor = +0.25`
- `marksman`: `+0.10`
- 主力艦subdoctrine: `+0.03`
- `safety_first`: `critical_receive_chance = -0.25`
- damage control 1～3: 各 `critical_receive_chance = -0.10`、`naval_critical_effect_factor = -0.10`
- `crisis_magician`: `naval_critical_effect_factor = -0.50`
- 最大艦経験値bonus: `naval_critical_effect_factor = -0.30`
- `naval_repair_support`: 最大 `naval_critical_effect_factor = -0.25`。実際の値は支援率でscale

## 14. 乱数が振られる場所

主要な独立乱数は少なくとも次の4段階に分かれる。

| 段階 | 役割 | Ghidra上の根拠 |
|---|---|---|
| 命中乱数 | 攻撃が当たるか | 海軍fire exchangeの通常命中処理 |
| クリティカル乱数 | 命中後にクリティカルになるか | 水上 `naval_fire_exchange_member.cpp:0x58d`、航空 `naval_fire_exchange_air.cpp:0x483` |
| 部位ゲート乱数 | クリティカルを持続部位へ変換するか | `naval_fire_exchange_member.cpp:0x2f5` |
| 部位選択乱数 | 候補部位の重み付き抽選 | `ship.cpp:0x70f` |

このため、命中・クリティカル・部位付与・弾薬庫などの具体的部位は、それぞれ別の抽選である。

## 15. 海軍事故の「クリティカル」は別系統

機雷事故と訓練事故にも `CRITICAL_HIT` というdefineがあるが、これは上記の海戦fire exchangeとは別の事故ダメージ倍率系である。

| 事故define | バニラ | MOD |
|---|---:|---:|
| `NAVAL_MINES_ACCIDENT_CRITICAL_HIT_CHANCES` | 14% | 14% |
| `NAVAL_MINES_ACCIDENT_CRITICAL_HIT_DAMAGE_SCALE` | 5倍 | 5倍 |
| `NAVAL_MINES_ACCIDENT_STRENGTH_LOSS` | 50 | 0 |
| `NAVAL_MINES_ACCIDENT_ORG_LOSS_FACTOR` | 0.5 | 0 |
| `TRAINING_ACCIDENT_CHANCES` | 2%／時 | 0 |
| `TRAINING_ACCIDENT_CRITICAL_HIT_CHANCES` | 30% | 0 |
| `TRAINING_ACCIDENT_CRITICAL_HIT_DAMAGE_SCALE` | 4倍 | 0 |

MODは訓練事故を無効化し、機雷事故の基礎STR／ORG損失も0へ変更している。これらを砲・魚雷・航空攻撃のクリティカル率や部位抽選へ混ぜてはいけない。

## 16. Ghidraで確認した主要関数

| アドレス | 役割 |
|---|---|
| `0x1418cfaf0` (`FUN_1418cfaf0`) | 軽砲・重砲・艦船魚雷の攻撃値、貫徹、基礎クリティカル率、汎用倍率を構築 |
| `0x1418cecb0` (`FUN_1418cecb0`) | 装甲／貫徹比の段階表lookup。補間なし |
| `0x1418d3420` (`FUN_1418d3420`) | 艦砲・艦船魚雷の命中ダメージ、最終クリティカルmodifier、部位／汎用倍率分岐 |
| `0x1418da3c0` (`FUN_1418da3c0`) | 海軍航空攻撃の集約ダメージと航空専用クリティカル式 |
| `0x1418d4500` (`FUN_1418d4500`) | 航空攻撃からの部位損傷ゲート |
| `0x140bc6a40` (`FUN_140bc6a40`) | 候補部位収集、重み付き抽選、即時ダメージ変形、instance追加 |
| `0x140bca3a0` (`FUN_140bca3a0`) | 部位の持続modifier取得と `naval_critical_effect_factor` による拡縮 |
| `0x140bce9d0` (`FUN_140bce9d0`) | 部位instanceを含む艦性能再計算 |
| `0x140e75810` → `0x140bc66b0` | 全回復時に全クリティカル部位を消去 |

## 最終整理

```text
艦砲:
  信頼性依存
  装甲／貫徹の段階表依存
  MODは低確率・高威力

艦船魚雷:
  信頼性非依存の基礎率
  装甲／貫徹を完全迂回
  MODは基礎60%、部位なし15倍

艦上攻撃機・基地航空隊:
  艦船魚雷defineを使わない
  信頼性依存
  実装上は実装甲を比較せずAP critical bonusを常時使用
  複数機の汎用倍率追加分は機数で割る

共通:
  クリティカル後に独立した部位ゲート
  部位成功時は汎用倍率と重複しない
  部位は重み付き抽選され、全回復まで持続する
  effect factorは一部の持続modifierだけに適用され、-100%超では符号反転する
```

## 17. 照合した主なデータファイル

```text
バニラ:
C:\Program Files (x86)\Steam\steamapps\common\Hearts of Iron IV\common\defines\00_defines.lua
C:\Program Files (x86)\Steam\steamapps\common\Hearts of Iron IV\common\units\critical_parts\00_critical_parts.txt
C:\Program Files (x86)\Steam\steamapps\common\Hearts of Iron IV\common\units\*.txt
C:\Program Files (x86)\Steam\steamapps\common\Hearts of Iron IV\common\units\equipment\modules\00_ship_modules.txt

innovative balance v13.00:
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\defines\00_defines.lua
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\units\critical_parts\00_critical_parts.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\units\*.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\units\equipment\modules\00_ship_modules.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\technologies\MTG_naval_Support.txt
```
