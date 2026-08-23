# HOI4 `Air Superiority Mission Efficiency` の実装解析

> **innovative balance v13.00 対応版**  
> 対象modifierは `air_superiority_efficiency`。MOD日本語表示は「制空作戦効率」、英語表示は `Air Superiority Mission Efficiency`。実行ファイルの関数アドレスは今回のGhidraプロジェクトを基準とする。

## 1. 結論

`Air Superiority Mission Efficiency` は、画面に表示される一般的な任務遂行率を上げるmodifierではない。

実装上は、**制空任務を実行中の航空隊の次の3つの戦闘statへ、同じ割合を加算する任務固有戦闘modifier**である。

```text
Air Attack
Air Defence
Agility
```

たとえば、ほかのmodifierを無視して、

```text
air_superiority_efficiency = 0.15
```

なら、制空任務中は次のようになる。

```text
Air Attack  × 1.15
Air Defence × 1.15
Agility     × 1.15
```

したがって、このmodifierは撃墜能力だけでなく、生存能力にも影響する。特にAir DefenceとAgilityの両方を上げるため、片側だけが保有している場合は、表示値以上に空戦交換比へ影響する可能性がある。

一方、次の値には直接掛からない。

- 画面に表示される一般的な任務効率
- 空戦へ投入される航空機数
- 航続距離不足などによる任務遂行率
- 敵航空機の発見率
- 航空機のSpeed
- 戦略地域の航空優勢スコア
- 海戦内の空母CAPによる直接迎撃

---

## 2. modifierの識別

スクリプト上の内部名は次のとおり。

```text
air_superiority_efficiency
```

対応する英語localizationは次のとおり。

```text
MODIFIER_AIR_SUPERIORITY_EFFICIENCY:
    "Air Superiority Mission Efficiency"
```

英語説明文も、任務遂行率ではなくdogfight能力を上げるmodifierとして説明している。

```text
Improved tactics for planes on air superiority resulting in increased ability in dogfights.
```

MODの日本語localizationでは次の名称へ置き換えられている。

```text
MODIFIER_AIR_SUPERIORITY_EFFICIENCY: "制空作戦効率"
```

実行ファイル内のmodifier登録処理 `FUN_140543c60` では、このlocalization tokenが内部modifier ID `0x0B` と対応している。

```text
MODIFIER_AIR_SUPERIORITY_EFFICIENCY → modifier ID 0x0B
```

---

## 3. 任務別modifierの選択

任務固有効率を選択する関数は、

```text
FUN_140f431f0（0x140f431f0）
```

である。

簡略化すると次の処理になっている。

```text
if (missionMask & 0x04)
    return air_intercept_efficiency

if (missionMask & 0x01)
    return air_superiority_efficiency

if (missionMask & 0x02)
    return air_cas_efficiency

if (missionMask & 0x110)
    return air_nav_efficiency

return 0
```

内部modifier IDとの対応は次のとおり。

| 任務判定 | modifier | 内部ID |
|---|---|---:|
| `missionMask & 0x01` | `air_superiority_efficiency` | `0x0B` |
| `missionMask & 0x04` | `air_intercept_efficiency` | `0x0D` |
| `missionMask & 0x02` | `air_cas_efficiency` | `0x0F` |
| `missionMask & 0x110` | `air_nav_efficiency` | `0x10` |

重要なのは、**戦闘機であること自体ではなく、実際にどの任務として処理されているか**でmodifierが選択されることである。

したがって、同じ戦闘機でも、

```text
制空任務 → air_superiority_efficiency
迎撃任務 → air_intercept_efficiency
```

となる。

複数の任務ビットが同時に渡された場合、関数の判定順では迎撃 `0x04` が制空 `0x01` より先に評価される。ただし、通常の空戦エントリでは、その時点で実行中の任務種別が保存されて渡される。

---

## 4. Air Attack・Air Defence・Agilityへの適用

通常空戦用stat配列を作る関数は、

```text
FUN_140f23c00（0x140f23c00）
```

である。

この関数が返す配列は次のように対応している。

```text
output[0] = final Air Attack
output[1] = final Air Defence
output[2] = final Agility
output[3] = Air Attack  / AIR_WING_MAX_STATS_ATTACK
output[4] = Air Defence / AIR_WING_MAX_STATS_DEFENCE
output[5] = Agility     / AIR_WING_MAX_STATS_AGILITY
```

まず機体設計・装備から基礎statを取得し、国家、航空隊、エース、地域などのmodifierを集計する。その後、`FUN_140f431f0` が返した任務固有効率を、Air Attack・Air Defence・Agilityの3つへ同じ値だけ加算している。

実数表記へ直すと、おおむね次の式になる。

```text
M = air_superiority_efficiencyの合計

FinalAirAttack
    = BaseAirAttack × (1 + OtherAttackModifiers + M)

FinalAirDefence
    = BaseAirDefence × (1 + OtherDefenceModifiers + M)

FinalAgility
    = BaseAgility × (1 + OtherAgilityModifiers + M)
```

ゲーム内部では `1.0 = 100000` の固定小数点で計算される。

### 4.1 独立乗算ではなく加算modifier

`air_superiority_efficiency` は、最終statへ別枠で乗算されるわけではない。それぞれのstat modifier合計へ加算される。

たとえば、既にAir Attackへ+20%があり、制空作戦効率+15%を追加する場合、

```text
誤: 1.20 × 1.15 = 1.38
正: 1 + 0.20 + 0.15 = 1.35
```

となる。

既存のstat補正を `X`、追加する制空作戦効率を `M` とすると、追加前に対する実際の相対上昇率は、

```text
(1 + X + M) / (1 + X)
```

である。

---

## 5. 撃墜計算への影響

通常空戦の撃墜計算本体は、

```text
FUN_140f4a790（0x140f4a790）
```

である。

撃墜期待値の主要部分を簡略化すると、次の形になる。

```text
期待撃墜数
∝ 攻撃機数 × 攻撃側Air Attack
  ÷ 防御側Air Defence
  × [1 + 速度ボーナス - 機動性ペナルティ]
```

機動性ペナルティは、攻撃側Agilityが防御側Agilityより低い場合に発生する。

innovative balance v13.00では、

```text
agilityPenalty
    = (min(DefenderAgility / AttackerAgility, 2.0) - 1)
      × 0.9
      × baseDamage
```

である。

したがって、制空作戦効率は撃墜計算へ次の3経路から入る。

1. 攻撃側Air Attackを増やし、与える撃墜数を増加させる。
2. 防御側Air Defenceを増やし、受ける撃墜数を減少させる。
3. Agility比を改善し、攻撃時のペナルティを減らすか、防御時に敵のペナルティを増やす。

Speedには掛からないため、速度比から作られる速度ボーナスは直接変化しない。

---

## 6. +15%を片側だけが持つ場合の例

同一性能の戦闘機同士で、片側だけが、

```text
air_superiority_efficiency = 0.15
```

を持つものとする。ほかのmodifierと速度ボーナスは無視する。

### 6.1 ボーナス側が攻撃するとき

Air Attackが1.15倍になるので、Agilityによる追加変化がなければ、撃墜期待値の攻撃部分は概ね、

```text
1.15倍
```

となる。

元々相手よりAgilityが低かった場合は、Agilityも1.15倍になるため、既存の機動性ペナルティが軽減される可能性がある。

### 6.2 ボーナス側が攻撃されるとき

Air Defenceが1.15倍になるので、防御値だけを考えると、被撃墜期待値は、

```text
1 / 1.15 = 0.869565...
```

となり、約13.0%減少する。

さらに、攻撃側Agilityが1.00、防御側Agilityが1.15なら、攻撃側には、

```text
(1.15 / 1.00 - 1) × 0.9
= 0.135
```

すなわち13.5%分の機動性ダメージ減衰が発生する。

両方を合わせると、

```text
(1 - 0.135) / 1.15
= 0.865 / 1.15
≒ 0.7522
```

となる。

この単純化した条件では、ボーナス側の被撃墜期待値は約24.8%減少する。つまり、片側だけが保有する場合、+15%という表示よりも空戦交換比への影響が大きくなり得る。

---

## 7. 両陣営が同じ値を持つ場合

両陣営の同一性能機が、どちらも+15%を持つ場合、

```text
攻撃側Air Attack  = 1.15倍
防御側Air Defence = 1.15倍
```

なので、撃墜式の、

```text
Air Attack / Air Defence
```

では相殺される。

Agilityも双方1.15倍なら、

```text
DefenderAgility / AttackerAgility
```

の比率も変化しない。

したがって、両陣営が同じ値を保有し、機体性能・ほかのmodifier・投入機数も同じなら、通常空戦の交換比はほぼ変化しない。片側だけが取った場合や、取得量に差がある場合に強く効くmodifierである。

---

## 8. 一般的な `Air Mission Efficiency` との違い

紛らわしいが、次の2つは別modifierである。

```text
air_superiority_efficiency
air_mission_efficiency
```

実行ファイル内でも、異なる内部IDで登録されている。

| 表示・内部名 | 内部ID | 処理上の役割 |
|---|---:|---|
| `air_superiority_efficiency` | `0x0B` | 制空任務中の戦闘stat補正 |
| `air_mission_efficiency` | `0x11` | 一般的な任務遂行率の補正 |

一般的な任務効率を計算・表示する関数は、

```text
FUN_140f3ca40（0x140f3ca40）
```

である。

航空隊UIの `AIRWING_MISSION_EFFICIENCY` 表示も、この経路を使用している。この関数はmodifier ID `0x11` の `air_mission_efficiency` を参照するが、`0x0B` の `air_superiority_efficiency` は参照しない。

したがって、制空作戦効率を取得しても、UI上の一般的な任務効率が+15ポイントされるわけではない。

---

## 9. 直接影響しない値

### 9.1 参加機数・任務遂行率

`air_superiority_efficiency` は、空戦に参加する航空機数を作る一般的な任務効率計算には入らない。

よって、航続距離、補給、航空基地混雑、天候などで決まる任務遂行率を直接改善しない。

### 9.2 発見率

敵機の発見に使われるmodifierとは別である。MODにも、次のような別modifierが存在する。

```text
air_superiority_detect_factor
air_interception_detect_factor
```

`air_superiority_efficiency` 自体が発見率計算へ直接入る参照は確認できない。

### 9.3 戦略地域の航空優勢スコア

`air_superiority_efficiency` の内部ID `0x0B` を固有modifierとして直接取得する実使用箇所を調べたところ、任務別modifierを選ぶ `FUN_140f431f0` だけだった。

また、`FUN_140f431f0` の呼び出し先を追っても、航空優勢スコアを直接増幅する経路はない。

したがって、戦略地域へ加算される航空優勢値を直接+15%するmodifierではない。

ただし、戦闘能力上昇によって敵を多く撃墜し、自軍機を多く残せば、その後の航空優勢比率には間接的に影響する。

### 9.4 Speed

Speed getterは、

```text
FUN_140f25d80（0x140f25d80）
```

である。この関数は `air_superiority_efficiency` を参照しない。

したがって、制空作戦効率で速度そのものや速度ダメージボーナスが増えることはない。

---

## 10. 周辺getterの確認

`FUN_140f431f0` は共通の任務別modifier selectorであるため、通常空戦stat以外のgetterからも呼ばれている。

確認できた呼び出し元は次のとおり。

```text
FUN_140f23c00  通常空戦用 Air Attack / Defence / Agility
FUN_140f24d60  航空燃料消費関連stat
FUN_140f26210  Naval Strike Agility
FUN_140f26580  Naval Attack
```

ただし、現在の実行経路を確認すると、次のようになっている。

- `FUN_140f24d60` の実際の呼び出しは任務適用引数へすべて0を渡しており、任務固有効率を適用する分岐へ入らない。
- `FUN_140f26210` と `FUN_140f26580` はNaval Strike経路で呼ばれ、その場合はNaval Strike系の任務ビットによって `air_nav_efficiency` が選ばれる。
- 通常の制空空戦では `FUN_140f23c00` が `air_superiority_efficiency` をAir Attack・Air Defence・Agilityへ適用する。

したがって、現在確認できる通常のゲーム経路では、`air_superiority_efficiency` が燃料消費や対艦攻撃を増やすとは扱わない。

---

## 11. 艦上戦闘機・海戦内CAPとの関係

艦上戦闘機には、少なくとも次の2系統の戦闘処理がある。

```text
1. 戦略地域における通常dogfight
2. 海戦中、来襲攻撃機へ行う直接CAP迎撃
```

### 11.1 通常dogfight

艦上戦闘機であっても、通常の戦略地域空戦へ制空任務として参加し、任務ビット `0x01` が渡されるなら、`air_superiority_efficiency` が適用される。

この場合は陸上基地戦闘機と同様、`FUN_140f23c00` でAir Attack・Air Defence・Agilityへ加算される。

### 11.2 海戦内の直接CAP

海戦中に空母戦闘機が来襲攻撃機を迎撃する直接CAP処理は、

```text
FUN_1418daad0（0x1418daad0）
```

の別経路である。

この処理は通常空戦の `FUN_140f23c00 → FUN_140f4a790` を使用せず、`air_superiority_efficiency` の内部ID `0x0B` も取得しない。

したがって、**制空作戦効率は海戦内の直接CAP迎撃には適用されない**。

海戦内CAPを強化する要素と、戦略地域の通常dogfightを強化する要素は分けて考える必要がある。

---

## 12. innovative balance内の定義箇所

MOD内では、少なくとも次の箇所で `air_superiority_efficiency` が付与されている。

```text
common/technologies/air_doctrine.txt
    air_skirmish = {
        air_superiority_efficiency = 0.15
    }
```

```text
common/doctrines/subdoctrines/air/air_fighter_aircraft_subdoctrines.txt
    extended_search_patterns = {
        air_superiority_efficiency = 0.10
    }

    air_subdoctrine_tactical_flexibility = {
        air_superiority_efficiency = 0.10
    }

    combat_routines = {
        air_superiority_efficiency = 0.15
    }

    decoy_tactic = {
        air_superiority_efficiency = 0.05
    }
```

ただし、旧式の航空教義、サブドクトリン本体、選択式rewardが同じファイル群に混在している。DLC・ゲームルール・サブドクトリン選択によって同時取得できないものがあるため、ファイル上の全値を単純に合算してはいけない。

---

## 13. 解析上の関数対応

| 関数 | 役割 |
|---|---|
| `FUN_140543c60` | localization tokenとmodifier IDの登録 |
| `FUN_140f431f0` | 任務ビットから任務固有効率を選択 |
| `FUN_140f23c00` | 通常空戦用の最終Air Attack・Defence・Agilityを計算 |
| `FUN_140f3ca40` | 一般的な航空任務効率を計算・表示 |
| `FUN_140f25d80` | 最終Speedを計算 |
| `FUN_140f40320` | 攻撃機数と攻撃値を敵航空隊へ配分 |
| `FUN_140f4a790` | 通常空戦の撃墜期待値と確率的丸めを計算 |
| `FUN_1418daad0` | 海戦内の空母戦闘機による直接CAP迎撃 |

処理全体を簡略化すると次のようになる。

```text
MODIFIER_AIR_SUPERIORITY_EFFICIENCY
    ↓ 内部modifier ID 0x0B
FUN_140f431f0
    ↓ 制空任務ビット0x01のとき選択
FUN_140f23c00
    ↓ 同じ値を3能力へ加算
Air Attack / Air Defence / Agility
    ↓
FUN_140f40320
    ↓
FUN_140f4a790
    ↓
通常空戦の撃墜数
```

---

## 14. 最終整理

`Air Superiority Mission Efficiency` は、名前だけを見ると「制空任務の参加率」を上げるように見えるが、実装上の性質は異なる。

```text
分類:
    国家modifierによる任務固有戦闘stat補正

発動条件:
    航空隊が制空任務として処理されること

直接上昇する値:
    Air Attack
    Air Defence
    Agility

撃墜計算への影響:
    与撃墜数増加
    被撃墜数減少
    Agility比改善

直接上昇しない値:
    一般的な任務効率
    参加機数
    発見率
    Speed
    航空優勢スコア
    海戦内の直接CAP能力
```

特に対人戦バランスでは、両陣営へ同量配布した場合はAir Attack／Defence／Agility比が相殺されやすい一方、片側だけが取得した場合は攻撃・防御・機動性の3か所へ同時に作用する。このため、研究・サブドクトリン選択による取得差が空戦交換比へ大きく現れやすいmodifierである。

