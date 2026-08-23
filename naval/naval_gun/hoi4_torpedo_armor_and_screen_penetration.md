# HOI4 魚雷・装甲無視・「魚雷の水上艦貫徹」解析

> **対象:** HOI4 1.17.5.2 系の `hoi4.exe` をGhidraで解析し、Workshop ID `820039020` の **innovative balance v13.00** を適用した場合のdefine／modifierを照合した。  
> 実行ファイル側の分岐構造はGhidra解析、数値はバニラおよびMODのローカルファイルを根拠とする。

## 結論

1. **艦船の `torpedo_attack` は、通常の装甲／貫徹計算を完全に迂回する。**
   - 「貫徹値が非常に高い」のではなく、内部で貫徹欄へ専用値 `-1.0` を入れ、通常砲の `weaponPiercing / targetArmor` 分岐を通らない。
   - したがって、同じ条件なら装甲0の艦と装甲100の艦で、魚雷1発の基礎ダメージ倍率および基礎クリティカル率は変わらない。
2. **Wikiの「魚雷攻撃は装甲を無視する」という中心的な説明は正しい。**
   - ただし「一切軽減不能」という意味ではない。魚雷専用の被ダメージ軽減、敵魚雷クリティカル率、一般海軍ダメージ補正、命中率、直衛、乱数などは別に作用する。
3. **「魚雷の水上艦貫徹」は装甲貫徹ではない。**
   - 内部名は `naval_torpedo_screen_penetration_factor`。
   - 敵の直衛を抜けて主力艦・空母側を標的候補にできる既存確率を相対倍率で増やす。
   - 魚雷の1発当たりダメージ、装甲無視能力、命中率、クリティカル率を直接増加させない。

## 1. 装甲無視の実装

魚雷・軽砲・重砲の攻撃パラメータを作る本体は `FUN_1418cfaf0`（`0x1418cfaf0`）。兵装種別は次の対応になっている。

| `weaponType` | 兵装 |
|---:|---|
| 0 | 軽砲／対潜系 |
| 1 | 魚雷 |
| 2 | 重砲 |

通常砲は、攻撃値・砲側貫徹・目標側装甲を取得して `NAVY_PIERCING_THRESHOLDS` 系の表へ渡す。

```text
gunAttack       = weapon attack
weaponPiercing  = weapon piercing
targetArmor     = target armor

damageMultiplier = lookupDamageByPiercingRatio(
    weaponPiercing / targetArmor
)

criticalMultiplier = lookupCriticalByPiercingRatio(
    weaponPiercing / targetArmor
)
```

一方、魚雷分岐は次の形になっている。

```text
attack               = torpedo_attack
piercingSentinel      = -1.0
baseCriticalChance    = COMBAT_TORPEDO_CRITICAL_CHANCE
fallbackCriticalMult  = COMBAT_TORPEDO_CRITICAL_DAMAGE_MULT
```

その後の共通処理は、次の条件で魚雷専用経路へ入る。

```text
if piercingSentinel == -1.0:
    damageMultiplier = 1 - targetTorpedoDamageReduction
else:
    damageMultiplier = armorPiercingCalculation(...)
```

重要なのは、魚雷では左側の条件が必ず真になるため、後半の装甲比較が短絡評価される点である。魚雷分岐には通常砲が呼ぶ二つの貫徹比テーブル参照もない。

### Ghidra上の主要位置

| アドレス | 内容 |
|---|---|
| `0x1418cff27` | 魚雷の貫徹欄へ `-100000`（固定小数点の `-1.0`）を格納 |
| `0x1418cff2f` | `COMBAT_TORPEDO_CRITICAL_CHANCE` を取得 |
| `0x1418cff3a` | `COMBAT_TORPEDO_CRITICAL_DAMAGE_MULT` を取得 |
| `0x1418d039f` | 貫徹sentinelを確認する共通分岐 |
| `0x1418d0438` | 魚雷専用の被ダメージ軽減modifierを取得 |
| `0x1418d045a` | `1 - naval_torpedo_damage_reduction_factor` を最終倍率欄へ格納 |

## 2. 装甲の代わりに魚雷へ効く防御値

### 2.1 敵魚雷の損害軽減

内部名:

```text
naval_torpedo_damage_reduction_factor
```

式は次の通り。

```text
torpedoDamageAfterDefense
  = torpedoDamageBeforeDefense × (1 - torpedoDamageReduction)
```

例:

| modifier | 最終ダメージ倍率 |
|---:|---:|
| `+0.10` | 0.90倍 |
| `+0.20` | 0.80倍 |
| `-0.15` | 1.15倍 |

innovative balance v13.00 では、海軍subdoctrine内に次の実値がある。

- 戦艦・重巡洋艦: `+0.10` の選択肢
- 駆逐艦: `-0.15` の選択肢

つまり、このMODでは装甲を上げても魚雷ダメージは減らないが、専用modifierなら直接増減する。

### 2.2 敵魚雷のクリティカル率

内部名:

```text
naval_torpedo_enemy_critical_chance_factor
```

魚雷の基礎クリティカル率へ、対象側のこの値が掛かる。

```text
torpedoCritChanceStage1
  = COMBAT_TORPEDO_CRITICAL_CHANCE
  × (1 + targetEnemyTorpedoCriticalChanceFactor)
```

この後、攻撃側・防御側の一般クリティカル補正等が別段階で合成される。値が負なら魚雷クリティカルへの防御になる。

```text
例: base 60%、専用modifier -10%
60% × (1 - 0.10) = 54%
```

MOD内でこの専用modifierを実際の艦・技術へ与える有効な記述は見当たらず、AI評価用weightとコメントだけだった。したがって通常は後述する60%が専用防御なしの出発点になる。

## 3. 魚雷の命中からダメージまで

### 3.1 発射間隔

| define | バニラ | innovative balance v13.00 |
|---|---:|---:|
| `BASE_GUN_COOLDOWNS` の魚雷要素 | 4時間 | **25時間** |

MODでは魚雷が大幅に低頻度・高威力化されている。`naval_torpedo_cooldown_factor` があれば、この基礎間隔を短縮できる。たとえばMOD独自特性 `ib_hit_chance` は潜水艦へ `-80%`、駆逐艦・軽巡洋艦へ `-52%` を与える。

### 3.2 直衛による標的層の制限

魚雷発射時は、まず `FUN_1418d1bf0`（`0x1418d1bf0`）が敵直衛側の二つの閾値を読み、主力艦側・空母側まで標的候補を広げられるかを決める。

- 直衛が抜けなければ、魚雷自体が消えるのではなく前方の直衛艦が候補になる。
- 直衛を抜ければ主力艦側が候補になる。
- さらに空母側の閾値も抜ければ空母・輸送船側まで候補を広げられる。
- その後、候補艦から艦種、残存HP、戦闘状態等を使った重み付き乱択を行う。

現行exeでは、この層判定は一つの決定論的PRNG値を二つの閾値と比較している。Wikiの「追加の検定」という説明は概念的には近いが、実装上は独立した乱数を二回振る形ではない。

### 3.3 命中率

命中率は概略次の構造。

```text
targetHitProfile
  = targetVisibility × 100
    / (targetSpeed × 0.5 + 20)

profileMultiplier
  = clamp(
      (targetHitProfile / torpedoWeaponHitProfile)^2,
      MIN_HIT_PROFILE_MULT,
      1
    )

finalHitChance
  = max(
      COMBAT_MIN_HIT_CHANCE,
      COMBAT_BASE_HIT_CHANCE
      × profileMultiplier
      × firingShipStatusMultiplier
      × torpedoHitChanceModifiers
      × otherCombatModifiers
    )
```

射撃艦のOrganization低下は、既存資料で解析した一次関数の倍率としてこの命中率へ入る。魚雷は兵装hit profileが大きいため、小型・高速・低可視性目標に特に当たりにくい。

| define | バニラ | innovative balance v13.00 |
|---|---:|---:|
| `COMBAT_BASE_HIT_CHANCE` | 10% | **20%** |
| `COMBAT_MIN_HIT_CHANCE` | 2% | **0.05%** |
| 魚雷 `GUN_HIT_PROFILES` | 100 | **205** |
| `MIN_HIT_PROFILE_MULT` | 0 | 0 |

命中判定は `FUN_1418d3420` 内でPRNG値と最終命中率を比較する。

### 3.4 非クリティカル時のダメージ

位置取り、攻撃側／防御側modifier、ダメージ乱数などをまとめて `M` とすると、概略は次の形。

```text
effectiveTorpedoAttack
  = torpedo_attack
  × M
  × (1 - targetTorpedoDamageReduction)

strengthDamage
  = effectiveTorpedoAttack
  × COMBAT_DAMAGE_TO_STR_FACTOR

organisationDamage
  = effectiveTorpedoAttack
  × (1 - targetCurrentStrength / targetMaxStrength)
  × COMBAT_DAMAGE_TO_ORG_FACTOR
```

| define | バニラ | innovative balance v13.00 |
|---|---:|---:|
| `COMBAT_DAMAGE_RANDOMNESS` | 0.5 | 0.5 |
| `COMBAT_DAMAGE_TO_STR_FACTOR` | 0.60 | **1.33** |
| `COMBAT_DAMAGE_TO_ORG_FACTOR` | 1.00 | **1.33** |

ここにも装甲値は登場しない。

## 4. 魚雷クリティカル

### 4.1 発生率

| define | バニラ | innovative balance v13.00 |
|---|---:|---:|
| `COMBAT_TORPEDO_CRITICAL_CHANCE` | 10% | **60%** |
| `COMBAT_TORPEDO_CRITICAL_DAMAGE_MULT` | 2倍 | **15倍** |

魚雷分岐は、通常砲にある次の要素を使わない。

- 砲側貫徹値
- 目標装甲値
- `NAVY_PIERCING_THRESHOLD_CRITICAL_VALUES`
- 通常砲の基礎クリティカル率 `COMBAT_BASE_CRITICAL_CHANCE`
- 通常砲の信頼性による基礎クリティカル率低下

したがって「装甲が高いほど魚雷クリティカルを防げる」という関係はない。

### 4.2 クリティカル後の処理

魚雷クリティカルが成立すると、まず特定部位の故障を発生させられるかという処理へ進む。この段階では目標艦の状態・信頼性系の値や、各クリティカル種別の個別倍率が関係する。

特定部位のクリティカルが成立しなかった場合は通常故障へフォールバックし、魚雷専用の `COMBAT_TORPEDO_CRITICAL_DAMAGE_MULT` がHP・Orgダメージへ掛かる。

したがって、MODの「15倍」は**すべての魚雷クリティカルが必ず最終15倍になる**という意味ではない。部位クリティカルを引いた場合はその種類固有の効果・倍率へ分岐する。一方、通常故障へ落ちたときの倍率はバニラ2倍からMOD15倍へ非常に大きく上げられている。

## 5. 「魚雷の水上艦貫徹」の正確な式

ローカライズは次の通り。

```text
MODIFIER_NAVAL_TORPEDO_SCREEN_PENETRATION_FACTOR
日本語: 魚雷の水上艦貫徹
英語:   Torpedo screen penetration
説明:   敵の直衛効率を変更する。
```

modifier IDは `0x1bb`。登録箇所 `0x1405c89d2` と、使用箇所 `FUN_1418d1bf0` の双方で一致した。

敵側のある直衛閾値を `S`、攻撃側の水上艦貫徹modifier合計を `P` とすると、コードは次を計算する。

```text
effectiveScreenThreshold
  = max(0, 1 - max(0, 1 - S) × (1 + P))
```

通常の `0 ≤ S ≤ 1` では、突破率で書くと理解しやすい。

```text
baseBreakthroughChance      = 1 - S
modifiedBreakthroughChance  = min(1, (1 - S) × (1 + P))
```

つまり `P = +20%` は**突破率へ1.20倍**であり、敵直衛率から20ポイントを直接引く効果ではない。

### +20% の例

| 元の直衛閾値 `S` | 元の突破率 | 補正後突破率 | 補正後の有効閾値 |
|---:|---:|---:|---:|
| 100% | 0% | 0% | 100% |
| 80% | 20% | 24% | 76% |
| 50% | 50% | 60% | 40% |
| 25% | 75% | 90% | 10% |
| 0% | 100% | 100% | 0% |

この結果から分かること:

- 敵が100%直衛なら、+20%を持っていても突破率は0%。完全直衛を強制的に崩す効果ではない。
- 敵の直衛が一部崩れた状態で初めて効く。
- 敵直衛が0%なら既に突破率100%なので追加効果はない。
- コードは主力艦側と空母側の二つの直衛閾値へ同じ変換を行う。

### innovative balance v13.00 内の主な付与値

| ファイル／要素 | 値 |
|---|---:|
| `common/ideas/japan.txt` | +20% |
| `common/ideas/navy_spirits.txt` | +5% |
| `navy_submarine_doctrines.txt` | +5% の付与箇所が2系統 |

同時に有効な値は加算集計され、最後に `1 + P` として突破率へ掛かる。たとえば合計+25%なら、80%直衛に対する突破率は `20% × 1.25 = 25%` となる。

## 6. このmodifierが変えないもの

`naval_torpedo_screen_penetration_factor` は次を変更しない。

- 魚雷の装甲無視そのもの
- `torpedo_attack`
- 1発当たりHP／Orgダメージ
- 魚雷命中率
- 魚雷クリティカル率
- 魚雷クリティカル倍率
- 発射クールダウン

効果は**命中判定より前の「どの戦列を標的候補にできるか」**だけに入る。魚雷を直衛艦へ当てる能力を高める値ではなく、直衛の後ろにいる高価値艦へ魚雷が向かう機会を増やす値である。

また、この処理は海戦Fire Exchange内の艦船兵装 `torpedo_attack` 用である。艦上攻撃機・陸上攻撃機の `naval_strike_attack` は別の航空攻撃経路を通るため、「魚雷の水上艦貫徹」は航空雷撃の装甲／直衛処理を強化する値ではない。

### 航空雷撃の数値単位と空母補正

航空機設計画面の「対艦攻撃」は、艦艇攻撃の性能集計時に次の変換を受ける。

```text
navalAttackInternal = displayedNavalAttack × 0.1 × mission/stat modifiers
```

たとえば対艦攻撃26は、補正前の内部攻撃値2.6として航空strike式へ入る。この航空経路では、攻撃元の空母が同じ海戦の参加艦として実際に発見された場合だけ `NAir.NAVAL_STRIKE_CARRIER_MULTIPLIER` を追加で掛ける。現在のバニラは10.0、innovative balance v13.00は1.00である。

```text
airStrikeAttack = displayedNavalAttack × 0.1
                × mission/stat modifiers
                × (sourceCarrierFoundInThisCombat ? carrierMultiplier : 1)
```

これはあくまで `naval_strike_attack` の航空ダメージ経路である。倍率が掛かっても、艦船魚雷の `torpedo_attack`、魚雷専用クリティカルdefine、装甲無視sentinel、`naval_torpedo_screen_penetration_factor` を使うようになるわけではない。

## 7. Wiki記述の判定

[HOI4 Wikiの海戦記事（ミラー）](https://hoi4.parawikis.com/wiki/%E6%B5%B7%E5%86%9B%E6%88%98%E6%96%97) は、魚雷が装甲を無視すること、魚雷の標的層が直衛効率で制限されること、魚雷の基礎クリティカル率と通常故障倍率が通常砲と別であることを説明している。[Defines一覧](https://hoi4.paradoxwikis.com/Defines) にも魚雷専用のクリティカル率・倍率defineが掲載されている。

Ghidra結果との比較:

| Wiki上の主張 | 判定 | 補足 |
|---|---|---|
| 魚雷は装甲を無視する | **正しい** | 高貫徹ではなく専用sentinelで装甲分岐を迂回 |
| 魚雷クリティカル率は通常砲と別 | **正しい** | 専用defineを直接使用 |
| 魚雷の通常故障倍率は信頼性で縮小しない | **正しい** | 専用倍率を直接格納 |
| 直衛を抜ける基礎確率は `1 - 直衛閾値` | **正しい** | 水上艦貫徹はその突破率を相対倍率化 |
| 空母側へは追加検定がある | **概念的には正しい** | 現行exeは同じPRNG値を二つの閾値へ比較 |

なお、ミラー記事は表示上1.14向けであるため、数値はそのまま現行バージョンへ流用すべきではない。本資料では1.17.5.2系exeの分岐を確認し、さらにMODの現在値を別途読み込んでいる。

## 8. 実戦上の要約

- **装甲艦を魚雷から守る目的で装甲だけを増やしても、魚雷1発のダメージ・クリティカル率は下がらない。**
- 防御手段は、直衛を100%近く維持して魚雷を前列へ吸わせること、魚雷専用被ダメージ軽減、敵魚雷クリティカル率低下、小型・高速・低可視性による命中回避である。
- **「魚雷の水上艦貫徹」は部分的に崩れた直衛へ追い打ちを掛ける能力。** 完全直衛への強制突破ではない。
- innovative balance v13.00 は魚雷を「25時間ごとの低頻度攻撃」にする代わり、基礎クリティカル率60%、通常故障時15倍、STR／Org変換1.33という非常に尖った設計にしている。
- このMODでは魚雷1発の期待値を考える際、装甲値よりも、直衛維持、魚雷命中率、専用軽減、クールダウン短縮、クリティカル分岐の方がはるかに重要である。
