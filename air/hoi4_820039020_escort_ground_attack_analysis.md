# HOI4 MOD 820039020：Escort Efficiency / Ground Attack Factor 内部計算調査

調査対象は Workshop MOD `820039020` のスクリプト・defineと、解析済み `hoi4.exe` の計算処理です。以下の式では、ゲーム内部の固定小数点 `100000 = 1.0` を通常の小数へ直して表記します。

## 結論

- `air_escort_efficiency` は、護衛に使える航空団が作る「妨害軽減スコア」の**最終倍率**である。`+10%` なら完成済み護衛スコアが `×1.10` になる。
- 護衛スコアは敵の総妨害スコアと比較され、`min(護衛 / 妨害, 1)` の割合だけ妨害を消す。したがって、護衛が十分なら妨害は0になり、それ以上の `escort efficiency` はその時点では無駄になる。
- `ground_attack_factor` は機体数や出撃効率を増やす係数ではなく、航空団が使用する**最終対地攻撃値**への倍率である。
- 同じ国家補正層にある `ground_attack_factor` 同士は加算してから一度だけ掛ける。例：`+5%` と `+3%` は `×1.08` であり、`×1.05 ×1.03` ではない。
- エース補正、天候補正、国家補正は別レイヤーなので、レイヤー間は乗算になる。
- 陸戦CASでは、最終対地攻撃値が `配分済み機数 × 対地攻撃値 × 0.10` という攻撃回数生成の前段に入る。このため `ground_attack_factor` は基本的にSTR・ORG両方の損害を同率で増やす。

---

## 1. Escort Efficiency

### 1.1 何を増やす係数か

内部登録IDは `0x0c`。航空団ごとの護衛・妨害軽減値を作る関数 `FUN_140f3b8c0` の**最後**で読み出され、次の形で掛かる。

```text
EscortPower
  = EscortPlaneCount
  × (1 + SpeedTerm     × ESCORT_SPEED_FACTOR)
  × (1 + AgilityTerm   × ESCORT_AGILITY_FACTOR)
  × (1 + AirAttackTerm × ESCORT_ATTACK_FACTOR)
  × (1 + Detection     × DISRUPTION_DETECTION_FACTOR)
  × ESCORT_FACTOR
  × (1 + air_escort_efficiency)
```

`SpeedTerm`、`AgilityTerm`、`AirAttackTerm` は航空団から計算される内部値である。係数名と各値を読み出す専用関数の対応から、項の種類を特定した。

このMODのdefineは次のとおり。

```lua
DISRUPTION_DETECTION_FACTOR = 0.0
ESCORT_FACTOR               = 2.0
ESCORT_SPEED_FACTOR         = 0.0
ESCORT_AGILITY_FACTOR       = 8.0
ESCORT_ATTACK_FACTOR        = 4.0
```

したがって、このMODでは速度と地域探知率の項が消え、実質的に次の式になる。

```text
EscortPower
  = 2 × EscortPlaneCount
  × (1 + 8 × AgilityTerm)
  × (1 + 4 × AirAttackTerm)
  × (1 + air_escort_efficiency)
```

重要なのは、`air_escort_efficiency` が護衛機数だけでなく、機動性・対空攻撃・基本係数まで含めた**完成スコア全体**を増幅する点である。

### 1.2 敵の妨害へどう適用されるか

戦略航空処理 `FUN_140bdcc40` は対象航空地域について、航空団から次を合計する。

```text
TotalDisruption          = Σ 各航空団の妨害スコア
TotalDisruptionReduction = Σ 各航空団のEscortPower
```

その後、次の比率を求める。

```text
ReductionRatio = min(TotalDisruptionReduction / TotalDisruption, 1.0)

EffectiveDisruption
  = RawDisruption × (1 - ReductionRatio)
```

つまり護衛効率は敵機を直接撃墜する補正ではなく、爆撃・攻撃側が受ける**有効妨害量を減らす**補正である。

### 1.3 上限と限界効用

`air_escort_efficiency` 自体を一定値で打ち止めにする処理は、護衛スコア生成式では確認できなかった。ただし、その後の `ReductionRatio` は最大 `1.0` に丸められる。

- 護衛スコア < 妨害スコア：追加の護衛効率が有効。
- 護衛スコア ≥ 妨害スコア：有効妨害は0。追加分はその戦闘状況では効果なし。
- 敵の妨害が0：比較処理自体に意味がなく、護衛効率の実益もない。
- 護衛適格フラグを持たない航空団：護衛スコアは0にされる。係数だけ取得しても護衛任務を発生させるわけではない。

### 1.4 数値例

補正前の護衛スコアを80、敵の妨害スコアを100とする。

```text
補正なし：ReductionRatio = 80 / 100 = 80%
           EffectiveDisruption = 20

+10%：     EscortPower = 80 × 1.10 = 88
           ReductionRatio = 88%
           EffectiveDisruption = 12
```

表示上は `+10%` だが、残る妨害は20から12へ減るため、この例では**残存妨害が40%減る**。護衛と妨害の差を埋める係数なので、100%軽減へ近いほど残存妨害に対する相対効果が大きくなる。

一方、補正前護衛スコアが95なら `+10%` 後は104.5だが、軽減率は100%で打ち止めになる。

### 1.5 このMOD内の主な付与元

| 付与元 | 値 |
|---|---:|
| 技術 `air_offense` | +17.5% |
| `fighter_bomber_coordination` | +10% |
| `escort_cooperation` | +10% |
| `escort_procedures` | +10% |
| medium aircraft subdoctrine 本体 | +10% |
| `battlefield_air_interdiction_spirit` | +5% |
| 空軍長特性 `air_air_superiority_1/2/3` | +2% / +4% / +6% |

国家補正プールに集約される同種の値は加算され、護衛式の最後で `(1 + 合計値)` として一度だけ掛かる。

### 1.6 混同しやすいもの

- `air_escort_efficiency` は航空戦の護衛。海軍の `convoy_escort_efficiency` とは別物。
- `air_mission_efficiency` は任務に実際参加できる機数・稼働率側。護衛スコアの最終倍率である本係数とは別段階。
- `ESCORT_FACTOR = 2.0` は全護衛にかかるdefine、本係数は国家・特性等から来る追加倍率。

---

## 2. Ground Attack Factor

### 2.1 `air_ground_attack`、`ground_attack`、`ground_attack_factor` の違い

内部では3つを分けて扱う。

| 名前 | 内部位置 | 役割 |
|---|---:|---|
| `air_ground_attack` | 航空機装備ステータス `0x36` | 機体設計・装備由来の基礎対地攻撃値 |
| `ground_attack` | modifier ID `0xe0` | 対地攻撃値への固定加算 |
| `ground_attack_factor` | modifier ID `0xe1` | 対地攻撃値への割合倍率 |

例えば基礎値20に `ground_attack = 2` と `ground_attack_factor = 0.10` が同じ国家段階で入れば、国家段階は `(20 + 2) × 1.10 = 24.2` となる。

なお、`category_cas = { air_ground_attack = 0.1 }` のような記述は航空機カテゴリの装備ステータス補正であり、`ground_attack_factor` とは別の基礎値側レイヤーである。

### 2.2 最終対地攻撃値の作成順

内部関数 `FUN_140f25180` は装備ステータス `0x36` を取得した後、概略として次の順で処理する。

```text
G0 = equipment-derived air_ground_attack

G1 = (G0 + AceGroundAttackFlat)
     × (1 + AceGroundAttackFactor)

G2 = G1
     × (1 + contextual ground_attack_factor)
     × (1 + weather ground_attack_factor)

FinalGroundAttack
   = (G2 + CountryGroundAttackFlat)
     × (1 + CountryGroundAttackFactor)
```

ここでの「contextual」は戦闘・地域側から渡される補正層をまとめた表現である。重要な確定事項は次の2点。

1. 各補正プールの中では値を合算して `(1 + 合計)` とする。
2. エース、天候、国家など別プールは順番に掛かるため、レイヤー間は乗算になる。

エース補正はエース専用取得処理を通るため、実値はエース有効度等による調整後の値になる。

### 2.3 陸戦CASの攻撃回数へ入る式

陸戦への航空攻撃処理 `FUN_141258900` では、戦闘へ配分済みの機数を `P`、前節の最終対地攻撃値を `G` とすると、まず次を計算する。

```text
ExpectedAttackRolls = P × G × 0.10
```

小数部分は捨てっぱなしではなく、固定小数点の端数を確率として使い、整数の攻撃回数へ確率丸めする。例：`210.4` 回なら210回に60%、211回に40%という形で、長期平均は210.4回になる。

各有効攻撃についてSTRダイスとORGダイスを振る。このMODでは次の値になっている。

```lua
LAND_AIR_COMBAT_STR_DICE_SIZE       = 1
LAND_AIR_COMBAT_ORG_DICE_SIZE       = 1
LAND_AIR_COMBAT_STR_DAMAGE_MODIFIER = 0.15
LAND_AIR_COMBAT_ORG_DAMAGE_MODIFIER = 0.10
```

ダイスサイズが1なので、有効攻撃1回あたりの基礎ダイスはSTR・ORGとも常に1。その後に戦闘側係数、敵の `CAS damage reduction`、ORG専用の `air_close_air_support_org_damage_factor` 等が入り、最後にSTRは0.15、ORGは0.10でスケールされる。

概略は次のとおり。

```text
STR damage
  ∝ AttackRolls × combat factors
    × (1 - CASDamageReduction)
    × 0.15

ORG damage
  ∝ AttackRolls × combat factors
    × (1 - CASDamageReduction)
    × (1 + CASOrgDamageFactor)
    × 0.10
```

したがって、他条件が同じなら `ground_attack_factor = +5%` は攻撃回数を約5%増やし、STR・ORG損害の期待値もともに約5%増やす。ORG専用係数を同時に持つ場合、ORGだけはさらに増える。

### 2.4 数値例

基礎対地攻撃20、戦闘参加機100機、その他の補正なしとする。

```text
補正なし：G = 20
ExpectedAttackRolls = 100 × 20 × 0.10 = 200

+5%：G = 20 × 1.05 = 21
ExpectedAttackRolls = 100 × 21 × 0.10 = 210
```

敵軽減等を無視した単純化では、define適用後の期待スケールは次のようになる。

```text
STR：200 × 0.15 = 30   → 210 × 0.15 = 31.5
ORG：200 × 0.10 = 20   → 210 × 0.10 = 21
```

同じ国家補正層に `+5%` と `+3%` がある場合：

```text
20 × (1 + 0.05 + 0.03) = 21.6
```

フル効果のエース `+8%` と国家補正 `+5%` は別レイヤーなので、単純化すると：

```text
20 × 1.08 × 1.05 = 22.68
```

### 2.5 このMOD内の主な付与元

| 付与元 | 値 |
|---|---:|
| `forward_air_controllers` | +5% |
| `air_subdoctrine_flexible_fire_support` | +5% |
| `massed_strike_spirit` | +5% |
| 空軍長特性 `air_close_air_support_1/2/3` | +1% / +2% / +3% |
| 支援機エース `support_good/unique/genius` | +3% / +5% / +8% |

空軍長のCAS特性は同時に `air_close_air_support_org_damage_factor` も同値だけ持つ。たとえばレベル3は、対地攻撃回数を約3%増やしたうえ、ORG損害側へさらに `+3%` の専用倍率を与える。単純化すればORGは `1.03 × 1.03 = 1.0609`、すなわち約+6.09%になる一方、STRは約+3%に留まる。

### 2.6 上限・効かなくなる条件

`ground_attack_factor` または最終対地攻撃値に対する明示的な上限は、確認した最終ステータス作成関数にはない。ただし実損害には別の制限がある。

- 戦闘参加機数は敵戦闘幅で制限される。MOD値は `LAND_AIR_COMBAT_MAX_PLANES_PER_ENEMY_WIDTH = 2.0`。
- 地上攻撃に参加できる航空団数は `COMBAT_MAX_WINGS_AT_GROUND_ATTACK = 10000`。このMODでは非常に高く、通常は事実上の制限になりにくい。
- 夜間は `CAS_NIGHT_ATTACK_FACTOR = 0.1`。夜間側の別処理で大きく弱体化される。
- 任務効率、航続距離、補給、天候、妨害、実際に戦闘へ割り当てられた機数は別段階。
- 敵の `cas_damage_reduction` は対地攻撃値作成後の損害を減らす。
- 攻撃対象・任務が対地攻撃値を使わない場合、当然ながら本係数は効果を持たない。対空攻撃、戦略爆撃、対艦攻撃を直接増やす係数ではない。

対地攻撃値の同じ取得関数は、陸戦CASのほか補給地点・鉄道を扱う航空任務処理からも呼ばれている。そのため `ground_attack_factor` は通常の陸戦支援だけでなく、対地攻撃値を参照する兵站攻撃系処理にも波及する。

---

## 3. 解析根拠

### MODファイル

- `common/defines/00_defines.lua`：護衛、妨害、陸戦航空ダメージ、参加上限、夜間係数
- `common/technologies/air_doctrine.txt`：`air_offense = air_escort_efficiency +0.175`
- `common/doctrines/subdoctrines/air/*.txt`：両係数のサブドクトリン付与元
- `common/ideas/air_spirits.txt`：空軍精神
- `common/country_leader/00_traits.txt`：空軍長特性
- `common/aces/00_aces.txt`：支援機エースの `ground_attack_factor`

### 実行ファイルで確認した主な位置

| 内容 | アドレス / 関数 |
|---|---|
| `air_escort_efficiency` パーサ | `0x1400a7977` |
| 同modifier登録ID `0x0c` | `0x140579eb1` |
| 航空団ごとの護衛スコア生成 | `FUN_140f3b8c0` |
| 総妨害・総軽減の比較、軽減率上限1.0 | `FUN_140bdcc40` |
| `ground_attack_factor` パーサ | `0x1400a7197` |
| 固定加算 `ground_attack` のID | `0xe0` |
| 倍率 `ground_attack_factor` のID | `0xe1` |
| 最終対地攻撃値の作成 | `FUN_140f25180` |
| 陸戦航空攻撃回数・STR/ORGダメージ | `FUN_141258900` |
| `cas_damage_reduction` のID | `0x162` |
| `air_close_air_support_org_damage_factor` のID | `0x13` |

## 4. 確度

- 係数ID、読み出し位置、乗算順序、護衛対妨害の比率上限、陸戦CASの `P × G × 0.10`、STR/ORG defineの使用：逆コンパイルで直接確認したため高確度。
- `SpeedTerm` 等の表示名：使用defineと航空団ステータス取得処理からの意味付け。式中の位置と係数対応は直接確認済み。
- UI表示上の丸めや、すべての任務選別条件の名称対応：今回の主対象ではないため完全列挙していない。

