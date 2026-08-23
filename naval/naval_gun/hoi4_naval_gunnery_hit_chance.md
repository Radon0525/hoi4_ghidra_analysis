# HOI4 艦砲・艦船魚雷の命中率・Organization・天候・乱数処理

> **innovative balance v13.00 対応版**  
> 対象: HOI4 1.17系の提供Ghidraプロジェクトと、Workshop ID `820039020`

## 1. 結論

海戦中の重砲・軽砲の命中率は、単純な「基礎命中率－回避値」ではない。対象exeでは、目標艦の現在速度と視認性から `TargetProfile` を作り、砲種別の `GunProfile` との比を二乗して基礎命中率へ掛ける。

```text
TargetProfile
  = EffectiveVisibility
  × HIT_PROFILE_MULT
  × HullHitProfileMult
  / (max(0.1, EffectiveSpeed × HIT_PROFILE_SPEED_FACTOR)
     + HIT_PROFILE_SPEED_BASE)

ProfileFactor
  = clamp(
      (TargetProfile / GunProfile)^2,
      MIN_HIT_PROFILE_MULT,
      1
    )

StatusFactor
  = max(0,
      1
      + (1 - OrgRatio)      × COMBAT_LOW_ORG_HIT_CHANCE_PENALTY
      + (1 - ManpowerRatio) × COMBAT_LOW_MANPOWER_HIT_CHANCE_PENALTY
    )

AccuracyFactor
  = 1
  + GeneralNavalHitChance
  + 砲種別の艦船装備命中補正
  + 砲種別の国家・提督・戦闘側補正
  + 対象国別命中補正

GeneralNavalHitChance
  = 国・提督・海軍精神等の naval_hit_chance
  + 艦船経験値による naval_hit_chance
  + 夜間の naval_hit_chance × Darkness
  + 天候の naval_hit_chance × WeatherScale

WeatherScale
  = ShipNavalWeatherPenaltyFactor
  - 1
  + clamp(1 + CountryNavalWeatherPenaltyFactor, 0, 1)

RawHitChance
  = COMBAT_BASE_HIT_CHANCE
  × ProfileFactor
  × StatusFactor
  × AccuracyFactor

FinalHitChance
  = max(COMBAT_MIN_HIT_CHANCE, RawHitChance)
```

最後に一様乱数を1回取得し、次を満たせば命中する。

```text
random < FinalHitChance
```

したがって命中率を直接左右する中心要素は、**目標の現在速度・現在視認性、射手のOrg・人員率、一般命中補正、砲種別命中補正**である。天候には、射手の `naval_hit_chance` を下げる直接経路と、目標艦を減速させて逆に当てやすくする間接経路が同時に存在する。

今回の再検証では、通常範囲の天候補正に明白な符号逆転や未適用は見つからなかった。ただし、表示上の「命中率+10%」は最終確率を1.10倍する値ではなく、他の命中補正と同じ括弧へ `+0.10` する値である。また悪天候は目標速度も下げる。この二点だけで、体感と表示が大きくずれる場合がある。

## 2. 主要関数

| 役割 | 関数 | アドレス |
|---|---|---:|
| 重砲・魚雷・軽砲の攻撃値と命中率を構築 | `FUN_1418cfaf0` | `0x1418cfaf0` |
| 水上艦／潜水艦のTargetProfileを計算 | `FUN_140bc9570` | `0x140bc9570` |
| 目標艦の現在有効速度を取得 | `FUN_140bcc2a0` | `0x140bcc2a0` |
| 輸送船の固定TargetProfileを取得 | `FUN_141a3e5d0` | `0x141a3e5d0` |
| 命中乱数を振り、命中後のダメージへ進む | `FUN_1418d3420` | `0x1418d3420` |
| 射撃側のOrg率を取得 | `FUN_140bcb030` | `0x140bcb030` |
| modifierを艦・国・提督・海域・天候等から集計 | `FUN_140bca3a0` | `0x140bca3a0` |
| 国家・ドクトリン側の天候倍率を作る | `FUN_140d0eda0` | `0x140d0eda0` |
| 重砲・軽砲等の攻撃値へ夜戦攻撃補正を入れる | `FUN_140bc6f90` | `0x140bc6f90` |

`FUN_1418cfaf0` の武器種引数は次の対応である。

```text
0 = 重砲
1 = 艦船魚雷
2 = 軽砲
```

## 3. MODの主要define

対象MODの `common/defines/00_defines.lua` では次の値になっている。

```text
COMBAT_BASE_HIT_CHANCE                 = 0.2
COMBAT_MIN_HIT_CHANCE                  = 0.0005
MIN_HIT_PROFILE_MULT                   = 0.0
COMBAT_LOW_ORG_HIT_CHANCE_PENALTY      = -0.5
COMBAT_LOW_MANPOWER_HIT_CHANCE_PENALTY = -0.25

GUN_HIT_PROFILES = {
  110.0,  # 重砲
  205.0,  # 魚雷
  130.0,  # 軽砲
}

CONVOY_HIT_PROFILE       = 120.0
HIT_PROFILE_MULT         = 100.0
HIT_PROFILE_SPEED_FACTOR = 2
HIT_PROFILE_SPEED_BASE   = 20
```

`COMBAT_BASE_HIT_CHANCE = 0.2` は20%、`COMBAT_MIN_HIT_CHANCE = 0.0005` は0.05%である。

## 4. 目標艦のhit profile

### 4.1 実装式

`FUN_140bc9570` は、対象艦について次を行う。

1. `FUN_140bcc2a0` で現在の有効速度を得る。
2. 対象状態に応じて水上視認性または潜水艦視認性を読む。
3. 現在の `navy_visibility` modifierを視認性へ掛ける。
4. 艦種の `hit_profile_mult` を掛ける。
5. 速度項と基礎値の和で割る。

正規化すると次になる。

```text
TargetProfile
  = EffectiveVisibility
  × HIT_PROFILE_MULT
  × HullHitProfileMult
  / (max(0.1, EffectiveSpeed × HIT_PROFILE_SPEED_FACTOR)
     + HIT_PROFILE_SPEED_BASE)
```

このMODでは戦艦、巡洋戦艦、空母、重巡、軽巡、駆逐艦、潜水艦の `hit_profile_mult` がすべて `1.0` である。通常の水上艦なら実質的に次の式となる。

```text
TargetProfile
  = SurfaceVisibility × 100
  / (max(0.1, CurrentSpeed × 2) + 20)
```

### 4.2 速度は設計画面の素の速度とは限らない

ここで使われるのは、艦船オブジェクトから取得した現在の有効速度である。艦船の `naval_speed` modifier、指揮官・海域等から来る速度補正、燃料不足補正を組み込む経路を通る。

ただしこのMODの `OUT_OF_FUEL_SPEED_FACTOR` は `-0.00` なので、少なくともdefine上は燃料切れによる速度低下が無効化されている。

### 4.3 二乗による効果

`TargetProfile` が砲のprofile未満で、上限・下限に触れていない範囲では、命中率は概ね次に比例する。

```text
HitChance ∝ EffectiveVisibility^2

HitChance ∝
  1 / (EffectiveSpeed × HIT_PROFILE_SPEED_FACTOR
       + HIT_PROFILE_SPEED_BASE)^2
```

したがって、視認性が2倍ならprofile部分の命中率は4倍になる。高速・低視認性を同時に実現すると、二乗式により非常に命中しにくくなる。

## 5. 砲種別profile補正

実装は次の順序でprofile補正を作る。

```text
ratio = TargetProfile / GunProfile
ProfileFactor = ratio^2
ProfileFactor = max(MIN_HIT_PROFILE_MULT, ProfileFactor)
ProfileFactor = min(1, ProfileFactor)
```

目標profileが砲profile以上になっても `1` で止まり、それ以上の命中ボーナスにはならない。

このMODでは `MIN_HIT_PROFILE_MULT = 0` なので、profile部分自体には最低保証がない。最後に全体へ `COMBAT_MIN_HIT_CHANCE = 0.0005`、すなわち0.05%の最低保証が掛かる。

## 6. MODにおける重砲と軽砲の逆転

GunProfileは値が小さいほど、同じ目標に対して命中しやすい。このMODでは次の設定である。

```text
重砲 = 110
軽砲 = 130
```

両方ともprofile減衰領域にある場合、砲種別の追加命中補正を除く重砲／軽砲の命中率比は次になる。

```text
(130 / 110)^2 ≈ 1.397
```

つまりprofileだけなら、重砲は軽砲より約39.7%当たりやすい。これは現在のバニラ値と逆である。

```text
現在のバニラ:
  重砲 = 70
  魚雷 = 100
  軽砲 = 45
```

バニラでは軽砲のprofileが小さく、小型高速艦へ当てやすい。MODでは重砲110、軽砲130へ変更され、profileだけを見ると重砲が優位になっている。

ただしMODの艦船モジュールには大きな砲種別命中補正がある。`00_ship_modules.txt` 内の個別値は、おおよそ次の範囲である。

```text
naval_light_gun_hit_chance_factor: +0.015 ～ +0.98
naval_heavy_gun_hit_chance_factor: -0.14  ～ +0.10
```

実行時には、設計から集計された最終艦船statが使われる。そのため実戦上は、次のような調整になっている可能性が高い。

```text
重砲:
  profile上の素の命中率は高め

軽砲:
  profile上の素の命中率は低めだが、モジュール固有命中補正で補う
```

## 7. 攻撃側の命中補正

`FUN_1418cfaf0` は一般命中補正から始め、武器種に応じた艦船装備statと戦闘側modifierを同じ加算括弧へ入れる。

```text
AccuracyFactor
  = 1
  + naval_hit_chance
  + WeaponHitChanceFactorFromShip
  + WeaponHitChanceFactorFromCombatSide
  + HitChanceAgainstTargetCountry
```

武器種別の対応は次である。

| 武器 | 艦船内stat offset | 対応するスクリプトstat |
|---|---:|---|
| 軽砲 | `ship + 0x240` | `naval_light_gun_hit_chance_factor` |
| 重砲 | `ship + 0x248` | `naval_heavy_gun_hit_chance_factor` |
| 魚雷 | `ship + 0x250` | `naval_torpedo_hit_chance_factor` |

これらは独立乗算ではなく、原則として `1 + 補正の合計` になる。

例えば、他の補正を無視して軽砲設計命中補正が `+0.94` なら、profile計算後の値へ `1.94` が掛かる。

## 8. Organizationと人員率

### 8.1 結論: 反比例ではなく一次関数

命中率へ入るのは**射撃側**のOrganization率と人員率である。目標側のOrgではない。二つは別々に乗算されず、同じ `StatusFactor` へ加算的に合成される。

```text
OrgRatio       = CurrentOrg / ModifiedMaxOrg
ManpowerRatio  = CurrentManpower / RequiredManpower

StatusFactor
  = max(0,
      1
      - 0.5  × (1 - OrgRatio)
      - 0.25 × (1 - ManpowerRatio)
    )
```

人員率100%なら次の一次関数になる。

```text
StatusFactor = 0.5 + 0.5 × OrgRatio
```

| 射撃側Org | 人員100%時の倍率 | 相対ペナルティ |
|---:|---:|---:|
| 100% | 1.000 | 0% |
| 80% | 0.900 | -10% |
| 75% | 0.875 | -12.5% |
| 50% | 0.750 | -25% |
| 25% | 0.625 | -37.5% |
| 20% | 0.600 | -40% |
| 0% | 0.500 | -50% |

Org 0%、人員0%では `1 - 0.5 - 0.25 = 0.25` になる。`0.5 × 0.75 = 0.375` ではない。

defineのコメントには「ORG is very low」とあるが、実装に低Org用の閾値分岐はない。Orgが100%から99%へ下がっただけでも倍率は100%から99.5%へ連続的に下がる。「very low」は最大ペナルティ側の説明であり、発動条件ではない。

### 8.2 getterと呼び出し経路

通常艦のOrg率は `FUN_140bcb030` が作る。

```text
currentOrg = *(ship + 0x5e0)
maxOrg     = FUN_140bc9ab0(ship)
orgRatio   = currentOrg × 100000 / maxOrg
```

内部では `1.0 = 100000` の固定小数点である。最大Orgは基礎値固定ではなく、`FUN_140bc9ab0` がmodifier込みで計算した現在の最大値である。輸送船は `FUN_141a3e620` が同様の比率を作る。

```text
FUN_1418d24e0
  └─ FUN_140bcb030            # 射撃側Org率

FUN_1418d24e0
  └─ FUN_1418d3100
       └─ FUN_1418d3420
            ├─ FUN_1418cfaf0  # StatusFactorを含む命中率
            └─ FUN_142187010  # 最終乱数
```

`FUN_1418d24e0` が射撃を行うFire Exchange MemberからOrg率を取得するため、補正対象が射撃艦であることも確認できる。

### 8.3 空母航空攻撃との区別

この `0.5 + 0.5 × OrgRatio` は、艦砲・艦船魚雷を発射する艦船Fire Exchange側の命中率へ掛かる。航空機の `naval_strike_targetting` へ同じ式を直接掛ける処理ではない。

空母自身のOrgは、海戦内航空攻撃では主として発艦効率・発艦機数側へ入る。

```text
艦砲・艦船魚雷:
  射撃艦Org → 命中率へ 0.5 + 0.5 × OrgRatio

艦載航空攻撃:
  母艦Org → 発艦効率・今回発艦できる機数へ概ね × OrgRatio
```

## 9. 攻撃側命中補正の全経路

### 9.1 modifier IDと加算先

`FUN_1418cfaf0` で最終的に同じ `AccuracyFactor` へ入る値は次のとおりである。

| 種類 | スクリプト名／内容 | 実行時modifier IDまたは艦offset | 合成 |
|---|---|---:|---|
| 一般命中 | `naval_hit_chance` | `0x28` | 加算 |
| 軽砲固有・艦設計 | `naval_light_gun_hit_chance_factor` | `ship + 0x240` | 加算 |
| 重砲固有・艦設計 | `naval_heavy_gun_hit_chance_factor` | `ship + 0x248` | 加算 |
| 魚雷固有・艦設計 | `naval_torpedo_hit_chance_factor` | `ship + 0x250` | 加算 |
| 軽砲固有・戦闘側 | 同名modifier | `0x1be` | 加算 |
| 重砲固有・戦闘側 | 同名modifier | `0x1bf` | 加算 |
| 魚雷固有・戦闘側 | 同名modifier | `0x1bd` | 加算 |
| 対象国別 | `naval_hit_chance_against...` 系 | `0x1c5` | 加算 |

一般命中 `0x28` は `FUN_1418d1500` から艦船用集計関数 `FUN_140bca3a0` へ渡り、艦、所有国、艦隊・提督、海域・州、経験、夜間、現在天候等の該当modifierを合計する。すべてが必ず存在するという意味ではなく、そのゲーム状態で有効なscopeだけが加わる。

対象MODで確認できる主な一般命中値は次のとおりである。

| 発生源 | 値 |
|---|---:|
| 海軍精神の一つ | `naval_hit_chance = +0.10` |
| 艦船経験値最大側 | `+0.30` |
| 艦船経験値最低側 | `-0.10` |
| 小雨・雪 | `-0.05` |
| 豪雨 | `-0.20` |
| 吹雪 | `-0.10` |
| 完全な夜間 | `-0.25` |

艦船経験値は最大・最低の端点用static modifierであり、実際には経験値段階に応じた値が使われる。

### 9.2 「命中率+10%」の正確な意味

`naval_hit_chance = 0.10` は、完成済みの最終命中率を `×1.10` する値ではない。`AccuracyFactor` の括弧へ `+0.10` する。

```text
設計・兵装補正なし:
  1.00 → 1.10
  最終命中率は相対 +10%

軽砲設計補正 +0.94:
  1.94 → 2.04
  相対増加 = 2.04 / 1.94 - 1 ≈ +5.15%
```

したがって大きな砲種別命中補正を持つ設計ほど、同じ表示上の `+10%` を追加したときの**相対的な伸び**は小さく見える。modifier自体が無効なのではない。

## 10. 天候・夜間の要約

天候・夜間を含む完全な命中式は本資料の第1節どおりである。重要点だけをまとめると次になる。

- `naval_hit_chance` は射手側の一般命中括弧へ入る。
- 悪天候の `naval_speed_factor` は目標も減速させ、速度・視認性profileの二乗式を通じて逆に当てやすくすることがある。
- `naval_weather_penalty_factor` は命中だけでなく、現在天候から取得する速度等のmodifierにも同じWeatherScaleを掛ける。
- `naval_night_attack` は命中率ではなく、夜間の兵装攻撃値を改善する。夜間命中ペナルティは別に残る。
- 通常値で `naval_hit_chance` の未適用や符号逆転は見つからなかった。ただしWeatherScale全体に下限clampがなく、軽減を100%超へ盛ると天候効果が反転し得る。

天候値、WeatherScaleの正確な合成、豪雨で晴天より命中率が上がる数値例、体感とずれる九つの原因、不具合監査は、横断的な検証として [hoi4_miscellaneous_reverse_engineering_notes.md](./hoi4_miscellaneous_reverse_engineering_notes.md) の「天候時の艦砲命中補正」節へ移した。

## 11. 基本数値例

以下は、Org・人員100%、天候・経験・設計命中補正なしの例である。

### 11.1 高速・低視認性艦

```text
速度       = 40
水上視認性 = 10

TargetProfile
  = 10 × 100 / (40 × 2 + 20)
  = 10
```

```text
重砲:
20% × (10 / 110)^2
≈ 0.165%

軽砲:
20% × (10 / 130)^2
≈ 0.118%
```

軽砲の最終設計命中補正が `+0.94` なら、他の補正を無視して次になる。

```text
0.118% × 1.94 ≈ 0.230%
```

### 11.2 大型・低速・高視認性艦

```text
速度       = 28
水上視認性 = 35

TargetProfile
  = 35 × 100 / (28 × 2 + 20)
  ≈ 46.05
```

```text
重砲命中率 ≈ 3.51%
軽砲命中率 ≈ 2.51%
```

## 12. 輸送船

輸送船は通常艦の速度・視認性式を使わず、`FUN_141a3e5d0` が `CONVOY_HIT_PROFILE` を直接返す。

```text
TargetProfile = 120
```

Org・人員100%、命中補正なしなら次になる。

```text
重砲:
20% × min(1, (120 / 110)^2)
= 20.00%

軽砲:
20% × (120 / 130)^2
≈ 17.04%

魚雷:
20% × (120 / 205)^2
≈ 6.85%
```

輸送船は重砲のprofileを上回るため、重砲のprofile減衰を受けない。

## 13. 旧evasion defineは実行時に使われない

defineには次の値と古い説明文が残っている。

```text
COMBAT_EVASION_TO_HIT_CHANCE              = 0.007
COMBAT_EVASION_TO_HIT_CHANCE_TORPEDO_MULT = 10.0
```

しかし対象exeで対応グローバルの全参照を調べると、どちらもdefine登録用関数からアドレスを渡す参照しかなく、実行時のREAD参照が存在しなかった。

| define | 対応グローバル | 参照状況 |
|---|---:|---|
| `COMBAT_EVASION_TO_HIT_CHANCE` | `0x14324a8b8` | 登録処理1件のみ、実行時READなし |
| `COMBAT_EVASION_TO_HIT_CHANCE_TORPEDO_MULT` | `0x14324a958` | 登録処理1件のみ、実行時READなし |

対照的に、次のhit profile関連グローバルには `FUN_1418cfaf0` または `FUN_140bc9570` から明確なREAD参照がある。

```text
MIN_HIT_PROFILE_MULT
GUN_HIT_PROFILES
HIT_PROFILE_MULT
HIT_PROFILE_SPEED_FACTOR
HIT_PROFILE_SPEED_BASE
CONVOY_HIT_PROFILE
```

したがって、このexeでは旧evasionコメント式を命中率計算へ含めてはいけない。現行実装はhit profile方式であり、二つのevasion defineは旧仕様の残骸と判断できる。

## 14. 最終乱数と射撃単位

`FUN_1418d3420` は `FUN_142187010` を呼び、次を判定する。

```text
random < FinalHitChance
```

乱数呼出しには、ビルド時のソース位置として次が残っている。

```text
C:\mnt\gsg\hoi4\hoi4_merged\hoi4\source\naval\naval_fire_exchange_member.cpp
0x575
```

重砲・軽砲の攻撃値は砲カテゴリ単位で集計され、射撃機会ごとに命中判定が行われる。`hg_attack` や `lg_attack` が大きくても、それ自体を命中率へ加算する処理ではない。これらは命中後の基礎ダメージを大きくする。

MODの基本射撃間隔は次である。

```text
MIN_GUN_COOLDOWN = 0.1時間

BASE_GUN_COOLDOWNS = {
  1.0,   # 重砲
  25.0,  # 魚雷
  1.0,   # 軽砲
}
```

## 15. 命中率へ直接入らない要素

最終命中率の構築関数を確認した限り、次は独立した直接因子として入らない。

- 艦隊の位置取り
- スクリーニング効率
- 戦列間距離・射程
- 目標の装甲と攻撃側の貫徹
- 信頼性
- 目標側のOrganization
- `hg_attack`／`lg_attack` の攻撃値そのもの

これらは、射撃可能性、目標選択、攻撃値、命中後のダメージ、装甲軽減、クリティカル、魚雷の標的層制限など別の作用点を持つ。

特に位置取りは攻撃・ダメージ側のペナルティや戦闘配置へ影響するが、`FinalHitChance` のprofile式へ `positioning` を直接掛ける命令はない。スクリーニングも砲弾1回のprofile命中率ではなく、主に目標層と魚雷経路へ作用する。

## 16. 潜水艦・爆雷の特例

潜水艦に対しては通常の水上砲撃と同じ扱いではない。

- 未発見状態では射撃処理から除外される。
- 重砲と艦船魚雷は潜水艦攻撃用として使われない。
- 発見された潜水艦へ軽砲カテゴリで攻撃する場合、爆雷用の攻撃値・profileへ切り替わる。

MOD値は次である。

```text
DEPTH_CHARGES_HIT_CHANCE_MULT = 1.1
DEPTH_CHARGES_DAMAGE_MULT     = 0.7
DEPTH_CHARGES_HIT_PROFILE     = 100.0
```

この場合は通常軽砲profileの130ではなく、爆雷profileの100が使われる。

## 17. バニラとの主要差

| define | 現在のバニラ | MOD v13.00 | 主な意味 |
|---|---:|---:|---|
| `COMBAT_BASE_HIT_CHANCE` | 0.10 | 0.20 | profile等を掛ける前の基礎率 |
| `COMBAT_MIN_HIT_CHANCE` | 0.005 | 0.0005 | 最終最低命中率 |
| 重砲profile | 70 | 110 | 大きいほど小profile目標へ不利 |
| 魚雷profile | 100 | 205 | 同上 |
| 軽砲profile | 45 | 130 | 同上 |
| `CONVOY_HIT_PROFILE` | 85 | 120 | 輸送船の固定profile |
| `HIT_PROFILE_SPEED_FACTOR` | 0.5 | 2 | 速度によるprofile低下の強さ |
| `HIT_PROFILE_SPEED_BASE` | 20 | 20 | 速度式の基礎値 |

MODは基礎命中率を10%から20%へ上げる一方、砲profileと速度係数を大幅に上げ、最低命中率を0.5%から0.05%へ下げている。

このため大型・高視認性・低速目標には高い基礎率が出やすいが、高速・低視認性艦にはバニラ以上に急激な命中率低下が起こりうる。最終的な重砲・軽砲差は、さらにモジュール固有の命中補正を加えて評価する必要がある。

## 18. 確度

### 高確度

- TargetProfileの速度・視認性式。
- 砲profile比を二乗し、0～1へclampする処理。
- MODの重砲110、魚雷205、軽砲130の順序。
- 基礎命中率20%、最低命中率0.05%。
- 射撃側Org・人員率の線形補正。
- 一般命中補正と砲種別命中補正が同じ加算factorへ入ること。
- 天候の `naval_hit_chance` と `naval_speed_factor` が同じWeatherScaleで調整されること。
- WeatherScaleが `ShipWeatherStat - 1 + clamp(1 + CountryModifier, 0, 1)` であること。
- 最終WeatherScale全体には当該ブロック内のclampがなく、極端な値なら符号反転し得ること。
- `naval_night_attack` が命中率ではなく武器攻撃値側へ、暗さに比例して入ること。
- 夜間命中ペナルティが天候軽減用ブロックとは別経路であること。
- 輸送船が固定profile 120を使うこと。
- 最終命中乱数の位置。
- 二つのevasion defineに実行時READ参照がないこと。

### 注意点

- `naval_hit_chance` や砲種別modifierは複数の国家、提督、艦船、戦闘環境から集計される。個別設計の最終値はゲーム状態によって変わる。
- 悪天候は射撃側の命中率と目標側の速度を同時に変えうるため、一つの表示modifierだけでは最終差を決められない。
- 整数固定小数点演算により、非常に小さい値では途中の丸めが入る。

### 主なアドレス

| アドレス | 内容 |
|---|---|
| `0x140bca3a0` | 艦船modifierの総合集計と天候寄与の調整 |
| `0x140bca5f5`～`0x140bca664` | 現在天候取得、WeatherScale作成、天候modifier加算 |
| `0x140d0eda0` | 国家側WeatherScaleを0～1へclamp |
| `0x140bcc2a0` | `naval_speed_factor` を含む現在有効速度 |
| `0x140bc9570` | 現在速度・視認性からTargetProfileを作成 |
| `0x140bcb030` | 通常艦の `CurrentOrg / ModifiedMaxOrg` |
| `0x140bc9ab0` | modifier込み最大Organization |
| `0x141a3e620` | 輸送船のOrg率getter |
| `0x1418cfaf0` | 命中率パラメータ計算本体 |
| `0x1418d0297`付近 | Orgペナルティ適用 |
| `0x140bc6f90` | `naval_night_attack` を武器攻撃値へ適用 |
| `0x1418d3420` | 射撃と命中乱数判定 |
| `0x1418d36e7` | 命中判定用PRNG呼び出し |
| `0x142187010` | 決定論的PRNG |

## 19. 照合した主なデータファイル

```text
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\defines\00_defines.lua
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\modifiers\00_static_modifiers.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\ideas\navy_spirits.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\country_leader\00_traits.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\unit_leader\00_traits.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\doctrines\subdoctrines\sea\navy_carrier_doctrines.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\doctrines\subdoctrines\sea\navy_screen_doctrines.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\doctrines\subdoctrines\sea\navy_submarine_doctrines.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\military_industrial_organization\policies\_navy_policies.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\units\equipment\modules\00_ship_modules.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\units\equipment\ship_hull_carrier.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\units\equipment\ship_hull_cruiser.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\units\equipment\ship_hull_heavy.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\units\equipment\ship_hull_light.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\units\equipment\ship_hull_submarine.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\units\battleship.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\units\battlecruiser.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\units\carrier.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\units\heavy_cruiser.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\units\light_cruiser.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\units\destroyer.txt
C:\Program Files (x86)\Steam\steamapps\workshop\content\394360\820039020\common\units\submarine.txt
```
