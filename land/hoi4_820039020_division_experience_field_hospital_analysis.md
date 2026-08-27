# HOI4「innovative balance」師団経験値・野戦病院の内部計算調査

調査対象：Workshop ID `820039020`、MODバージョン `13.00`、対応表記 `1.17.5.2`  
実行ファイル：ローカルの `hoi4.exe` を Ghidra で静的解析  
調査日：2026-08-25

## 結論

師団経験値を長期的に高く保つには、単に長時間戦わせるだけでは足りない。実コード上は次の三つの収支で決まる。

```text
師団経験値の増加
  = 実際に戦闘へ参加した時間による増加
  + 演習による増加（Regularまで）

師団経験値の減少
  = manpower/strength損失時に失われる「経験の総量」
  + 補充兵が入ることによる平均経験値の希釈
  + 編制変更時の経験値drop
```

野戦病院の二つの値は別々に働く。

- `casualty_trickleback`：師団から失われたmanpowerの一部を国家manpower poolへ戻す。
- `experience_loss_factor`：manpower損失時に師団の内部経験値poolから失われる量を減らす。

`casualty_trickleback` は、その場で師団のstrength低下を打ち消す値ではない。師団からは全損失人数が一度抜け、回収分が国家poolへ戻る。その後、通常のreinforcementで師団へ再投入される。

また、経験値低下式は `casualty_trickleback` を参照しない。したがって、人的資源回収と経験維持は相乗的には有用だが、コード上は独立した二効果である。

## 1. 経験値レベルとこのMODの戦闘補正

このMODの境界値は次のとおり。

```lua
UNIT_EXP_LEVELS = { 0.10, 0.25, 0.49, 0.81 }
EXPERIENCE_COMBAT_FACTOR = 0.15
ARMY_EXP_BASE_LEVEL = 1
```

実行コード `FUN_140c0b540` (`0x140c0b540`) は、現在値がいくつの境界を越えたかを数えてレベルを決める。`FUN_140c2e7b0` (`0x140c2e7b0`) は次の形で戦闘補正を作る。

```text
combat experience modifier
= (experience_level - Trained_level) × EXPERIENCE_COMBAT_FACTOR
```

よってこのMODでは：

| 内部値 | Level | 戦闘補正 |
|---:|---|---:|
| `< 0.10` | Green | -15% |
| `0.10 ～ < 0.25` | Trained | 0% |
| `0.25 ～ < 0.49` | Regular | +15% |
| `0.49 ～ < 0.81` | Seasoned | +30% |
| `>= 0.81` | Veterans | +45% |

添付Wikiのvanilla値である1段階25%ではなく、このMODは **1段階15%** である。それでもRegularとSeasonedの差は攻防双方に15ポイントあるため、長期戦で経験値を維持する価値は大きい。

## 2. 内部では「平均値」だけを保存していない

実コードは、概念的には次の二つを分けて扱う。

```text
experience pool = 師団が保有する経験の総量
displayed average experience = experience pool / 現在のmanpower
```

`FUN_1406d8720` (`0x1406d8720`) は、師団オブジェクトの `+0x518` にある経験値poolを現在manpowerで割って、レベル判定に使う平均経験値を返す。

このため、戦闘損失と補充は一つの出来事に見えても、内部的には次の順で効く。

1. strength/manpower損失が発生する。
2. 経験値poolから、後述する損失率分が除かれる。
3. 師団がunderstrengthの間は、残存兵だけで平均を取る。
4. reinforcementで新兵が入るとmanpower側の分母が増え、平均経験値が薄まる。

つまり「補充が来た瞬間だけに経験値損失イベントがある」という実装ではない。損失時に経験値poolを減らし、補充時には平均値の希釈として表面化する構造である。

## 3. 戦闘で経験値が増える条件

主要経路は次のとおり。

```text
FUN_140c3c5b0  戦闘中の時間更新
  -> FUN_140c22750
    -> FUN_140c23c80  combat-hour基礎経験値
  -> FUN_140c1de40    gain modifier適用後、経験値poolへ加算
```

基礎defineは：

```lua
UNIT_EXPERIENCE_PER_COMBAT_HOUR = 0.0001
UNIT_EXPERIENCE_SCALE = 1.0
```

満充足に近い通常状態なら、平均経験値換算の基礎は概ね **戦闘1時間あたり0.0001** である。内部では師団規模・現在manpowerに合わせて経験値poolへ変換して加えるため、大師団だけが平均経験値を不当に速く上げる構造ではない。

コードは戦場に登録されているだけでなく、実際にactive combatへ参加しているかも確認する。reserveに待機して攻撃を交換していない時間は、同じようには稼げない。

その後 `FUN_140c3a2e0` (`0x140c3a2e0`) が、おおむね次の形でgain modifierを適用する。

```text
final gain
= (base gain + flat unit experience gain)
   × (1 + experience_gain_army_unit_factor)
```

したがって `experience_gain_army_unit_factor` は、Seasoned/Veteransへ上げる時間を短縮する直接的な値である。このMODには陸軍長官・Army Spirit・ideaなどに正負両方の値が存在する。

### 3.1 理論上の連続戦闘時間

損失、充足低下、gain modifierを無視した理想値は：

| 区間 | 必要経験値 | 必要active combat時間 |
|---|---:|---:|
| Regular 0.25 -> Seasoned 0.49 | 0.24 | 2,400時間 = 100日 |
| Seasoned 0.49 -> Veterans 0.81 | 0.32 | 3,200時間 = 約133.3日 |
| Regular 0.25 -> Veterans 0.81 | 0.56 | 5,600時間 = 約233.3日 |

実戦では同時にcasualtyによる経験値損失が起きるため、これは下限に近い。高火力戦へ連続投入すると「戦闘時間では増えているのに、損失と補充で平均値が下がる」ことが普通に起こる。

## 4. 演習で経験値が増える条件

このMODでは：

```lua
TRAINING_MAX_LEVEL = 2
UNIT_EXP_LEVELS[TRAINING_MAX_LEVEL - 1] = 0.25
TRAINING_MIN_STRENGTH = 0.10
TRAINING_ATTRITION = 0.05
OUT_OF_FUEL_TRAINING_XP_GAIN_MULT = 0.0
```

実コード `FUN_140c23e60` (`0x140c23e60`) は、演習による師団経験値を **Regular境界の0.25まで** に制限する。SeasonedとVeteransは演習だけでは作れず、実戦が必要である。

重要なのは、師団自身の演習経験値が単純な固定 `+0.0015/day` ではないこと。この関数は概念的に次の値を作る。

```text
ideal daily training gain
~= Regular境界値0.25 / effective division training_time

actual daily gain
= ideal daily training gain
   × manpower充足側の係数
   × equipment/strength側の係数
   × supply/fuel等の係数
   × unit experience gain modifiers
```

したがって、Trained 0.10からRegular 0.25までの理想日数は、おおむね：

```text
(0.25 - 0.10) / (0.25 / effective_training_time)
= 0.60 × effective_training_time
```

例として実効training timeが120日なら約72日、180日なら約108日である。充足不足、補給不足、装備不足、燃料切れがあればさらに遅くなる。

`UNIT_EXPERIENCE_PER_TRAINING_DAY = 0.0015` は実行ファイル中では別の `FUN_140c228c0` (`0x140c228c0`) が直接読む。この経路は演習から国家のArmy XPを発生させる側であり、このMODは：

```lua
TRAINING_EXPERIENCE_SCALE = 0.0
```

なので、その国家Army XP gainを無効化している。**国家Army XPが0でも、師団自身の経験値gainまで0になったわけではない。**

通常のtraining attritionは主にequipment損耗であり、それだけで師団経験値poolを直接削る処理ではない。ただし装備・strength不足になれば、演習gainが低下または停止する。

## 5. manpower損失時の経験値低下式

中核は `FUN_140c3bbc0` (`0x140c3bbc0`) である。正規化すると：

```text
effective experience loss coefficient
= max(
     0,
     EXPERIENCE_LOSS_FACTOR
     + division/subunit experience_loss_factor
     + country/leader/other experience_loss_factor
   )

new experience pool
= old experience pool
   × (1 - manpower_loss_fraction
          × effective experience loss coefficient)
```

このMODの基礎値は：

```lua
EXPERIENCE_LOSS_FACTOR = 0.75
```

野戦病院なしで師団が10%相当のmanpower/strengthを失った場合：

```text
experience pool retention
= 1 - 0.10 × 0.75
= 0.925
```

つまり経験値poolは7.5%減る。20%損失なら15%減る。

`experience_loss_factor` は基礎0.75へ**加算**される。野戦病院Iの `-0.15` は「経験値損失を15%相対減」ではなく、係数を `0.75 -> 0.60` にする。合計が負になった場合は0へclampされるため、損失を経験値gainへ反転させることはできない。

## 6. 野戦病院I～IVの実値

通常の `field_hospital` のbase値は：

```text
casualty_trickleback = +0.20
experience_loss_factor = -0.15
```

II～IVは各段階でさらに：

```text
casualty_trickleback = +0.10
experience_loss_factor = -0.10
```

を累積加算する。Doctrine、idea、陸軍長官などを除いた技術系列だけなら：

| 段階 | casualty_trickleback | experience_loss_factor | 実効XP損失係数 | 病院なし比のXP損失削減 |
|---|---:|---:|---:|---:|
| なし | 0% | 0% | 0.75 | 0% |
| Field Hospital I | 20% | -15% | 0.60 | 20.0% |
| Field Hospital II | 30% | -25% | 0.50 | 33.3% |
| Field Hospital III | 40% | -35% | 0.40 | 46.7% |
| Field Hospital IV | 50% | -45% | 0.30 | 60.0% |

「病院IVはXPを45%維持する」ではない。基礎係数を0.75から0.30へ落とすため、病院なしと比べた実際のXP損失量は **60%減る**。

### 6.1 20%の損失を受けた例

満充足から20%相当を失い、その後元のmanpowerまで補充されたと仮定したとき、元の経験値に対する最終保持率は：

| 段階 | 経験値保持率 |
|---|---:|
| なし | 85% |
| Field Hospital I | 88% |
| Field Hospital II | 90% |
| Field Hospital III | 92% |
| Field Hospital IV | 94% |

VeteransやSeasonedのように元の経験値が高いほど、同じ割合差でも保存される絶対経験値は大きい。

### 6.2 casualty_tricklebackの実処理

人的資源損失側は `FUN_140bf9d10` (`0x140bf9d10`) で処理される。概念的には：

```text
raw manpower removed from division = strength damageから算出した人数
returned manpower = raw removed × casualty_trickleback
permanent casualties = raw removed - returned manpower
```

例としてraw lossが1,000人なら：

| 段階 | 国家manpower poolへ戻る | 恒久損失 |
|---|---:|---:|
| なし | 0 | 1,000 |
| Field Hospital I | 200 | 800 |
| Field Hospital IV | 500 | 500 |

師団のその場のstrengthは1,000人分落ちる。回収された500人が即座にその師団へ残るわけではなく、国家poolへ戻った後にreinforcement対象となる。

### 6.3 このMOD固有の追加効果

通常野戦病院は `category_all_infantry` に対して `max_strength = +0.10` も持つ。これは同じ被弾量に対する相対的なstrength/manpower損失を抑える方向へ働くため、上記二つとは別の間接的な経験維持効果もある。

## 7. Helicopter Field Hospital

このMODの `helicopter_field_hospital` のbase値は：

```text
casualty_trickleback = +0.25
experience_loss_factor = -0.10
category_all_infantry max_strength = +0.05
category_all_infantry default_morale = +0.10
```

基礎だけを比較すると：

- 人的資源回収：通常Iの20%より良い25%。
- 経験値維持：通常Iの実効0.60より悪い0.65。
- infantry max_strength：通常Iの+0.10より小さい+0.05。
- org recovery側：`default_morale +0.10`を持つ。

通常のField Hospital II～IV技術は、データ上 `field_hospital = { ... }` だけを強化し、`helicopter_field_hospital` には加算しない。helicopter型は専用subdoctrineから追加の `casualty_trickleback +0.05` などを受けられるが、通常病院の技術系列をそのまま継承するわけではない。

また両者は `same_support_type` で排他的に扱われるため、通常病院とhelicopter病院を同じ師団に入れて効果をstackすることはできない。

## 8. 編制変更による経験値低下

このMODのdefineは：

```lua
BATALION_NOT_CHANGED_EXPERIENCE_DROP = 0.0
BATALION_CHANGED_EXPERIENCE_DROP = 0.5
```

実コード `FUN_140c260f0` (`0x140c260f0`) は、旧編制のsubunit typeと個数を新編制と照合し、概念的に次を計算する。

```text
retained experience
= old experience
   × [ unchanged_ratio × (1 - 0.0)
       + changed_ratio × (1 - 0.5) ]
```

したがって、編制を少し変えただけで常に全経験値が50%消えるわけではない。**変更された旧battalion相当部分だけが50%drop扱い**になる。

例：旧編制10個のうち9個が同type・同数として残り、1個だけ別typeへ変わった場合：

```text
retention = 0.9 × 1.0 + 0.1 × 0.5 = 0.95
```

師団経験値全体の低下は5%。一方、旧battalionを全面的に別typeへ置き換えると保持率は50%になる。高経験値師団の大規模改編は、戦闘損失なしでも大きなXP低下を起こす。

## 9. 何をすると経験値が減るか

### 直接減る

- 戦闘でstrength/manpowerを失う。
- manpowerを除去する一部の特殊処理を受ける。
- 旧battalion typeを別typeへ置換・削除するような編制変更を行う。

### 平均値が薄まる

- 戦闘損失後にreinforcementを受ける。
- 低経験の人員・師団と統合し、経験値poolが平均化される。
- manpowerを大きく増やす編制へ変更し、新しい人員で平均が希釈される。

### 原則として直接は減らない

- organizationだけを失う。
- equipmentだけをattritionで失う。
- 通常の移動やstrategic redeploymentをする。
- Regular未満の師団を演習させる。

ただしequipment不足、補給不足、燃料切れは演習gainを止めたり、戦闘で損失しやすくしたりするため、間接的には経験維持を難しくする。

## 10. 長期戦でSeasoned/Veteransを作る実用方針

1. 新師団はまず演習でRegular 0.25まで上げる。Regular以降を演習し続けても師団levelは上がらない。
2. 高経験値化したい師団は、reserve待機ではなく実際にactive combatへ参加させる。
3. 高損害攻勢へ連続投入しない。経験値gainは時間比例だが、lossはmanpower損失割合に比例する。
4. strengthが大きく削れる前に交代し、補給・装備・燃料を回復させる。
5. `experience_gain_army_unit_factor` を取りつつ、`experience_loss_factor` も下げる。片側だけより収支が良い。
6. 高経験値師団ほどField Hospital III～IVの価値が高い。低経験の量産師団ではsupport slotとの比較が必要。
7. 高経験値になった後の全面的なtemplate conversionを避ける。必要なら低経験時に済ませるか、変更battalion比率を小さく刻む。
8. `casualty_trickleback` はnational manpower節約、`experience_loss_factor` は師団XP保存として別々に評価する。

要するに、Veteransを作る最短路は「長く戦わせる」ではなく、**Regularまで事前演習し、低損害でactive combat時間を稼ぎ、病院とloss modifierで補充希釈を抑え、編制を途中で大改造しないこと**である。

## 11. 主な根拠

### MODデータ

- `820039020/common/defines/00_defines.lua`
  - `UNIT_EXPERIENCE_PER_COMBAT_HOUR`
  - `TRAINING_MAX_LEVEL`
  - `UNIT_EXP_LEVELS`
  - `EXPERIENCE_COMBAT_FACTOR`
  - `EXPERIENCE_LOSS_FACTOR`
  - `BATALION_CHANGED_EXPERIENCE_DROP`
- `820039020/common/units/field_hospital.txt`
- `820039020/common/technologies/support.txt`
- `820039020/common/doctrines/subdoctrines/land/`

### 実行コード

| 役割 | 関数 |
|---|---|
| 経験値level判定 | `FUN_140c0b540` (`0x140c0b540`) |
| level別combat modifier | `FUN_140c2e7b0` (`0x140c2e7b0`) |
| combat-hour基礎gain | `FUN_140c23c80` (`0x140c23c80`) |
| training gain | `FUN_140c23e60` (`0x140c23e60`) |
| gain modifier適用 | `FUN_140c3a2e0` (`0x140c3a2e0`) |
| 経験値pool加算 | `FUN_140c1de40` (`0x140c1de40`) |
| manpower損失時のXP低下 | `FUN_140c3bbc0` (`0x140c3bbc0`) |
| casualty trickleback | `FUN_140bf9d10` (`0x140bf9d10`) |
| template変更drop | `FUN_140c260f0` (`0x140c260f0`) |

