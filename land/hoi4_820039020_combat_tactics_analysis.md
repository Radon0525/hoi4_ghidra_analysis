# HOI4「innovative balance」Combat Tactic総合調査

調査対象：Workshop ID `820039020`、MODバージョン `13.00`、対応表記 `1.17.5.2`  
主データ：`common/combat_tactics.txt`、`common/technologies/land_doctrine.txt`  
実行ファイル：ローカルの `hoi4.exe` を Ghidra で静的解析  
調査日：2026-08-25

## 結論

Combat Tacticは、12時間ごとに両軍が一つずつ引くweighted randomである。単なる一時的damage補正だけでなく、次の四つを変える。

1. 両軍のdamage dealing factor
2. 戦闘中のattacker movement speed
3. 戦場のcombat width
4. Battle Phaseと、次回以降に抽選できる戦術集合

このMODには53戦術、5種類の特殊phase、15本の `countered_by` 関係がある。

```text
候補になる
  -> active/unlock、攻防側、phase、地形、編成、指揮官条件を満たす
  -> scripted base weightを計算
  -> Preferred Tactic補正を掛ける
  -> 後手側のcounter候補ならscore差補正を掛ける
  -> 全候補からweight比例で一つ抽選
```

`base = { factor = 4 }` は4%ではない。他候補との相対weightが4という意味である。またPreferred Tacticもcounterも確定選択ではなく、weightを増やすだけである。

## 1. 戦術選択の実コード

### 1.1 更新間隔

`FUN_141265af0` (`0x141265af0`) は、戦術未選択時と戦闘hour counterが `TACTIC_SWAP_FREQUENCEY` の倍数になったときに再抽選する。

```lua
TACTIC_SWAP_FREQUENCEY = 12
```

したがって新規戦闘の開始時と、その後12時間ごとに戦術を選び直す。

### 1.2 使用可能候補

`FUN_14126a1e0` (`0x14126a1e0`) は全戦術を走査し、概ね次を順に確認する。

- `active = yes`、またはその国にその戦術が使用可能として登録されているか
- 現在のBattle Phase
- attacker用かdefender用か
- `trigger` の地形、river、編成、reserve、指揮官skill・traitなど
- scripted `base` が正のweightを返すか

`active = no` の戦術は、script effectの `unlock_tactic` / `enable_tactic` で使用可能にする必要がある。このMODではその大部分がLand Doctrine技術に結び付いているため、「解禁戦術」というより「採用したDoctrine routeから戦術poolへ追加される戦術」と捉えるほうが正確である。

### 1.3 weightの基本式

概念的な最終weightは次の積になる。

```text
W_final
= W_scripted_base
 × country preferred補正
 × army general preferred補正
 × field marshal preferred補正
 × counter候補補正
```

このMODのPreferred Tactic defineは：

```lua
COUNTRY_PREFERRED_TACTIC_WEIGHT_FACTOR      = 0.25
ARMY_GENERAL_PREFERRED_TACTIC_WEIGHT_FACTOR = 0.50
FIELD_MARSHAL_PREFERRED_TACTIC_WEIGHT_FACTOR = 0.25
```

よって該当すると、それぞれ通常weightの `×1.25`、`×1.50`、`×1.25` になる。同じ戦術へ三つとも重なる単純な場合は：

```text
1.25 × 1.50 × 1.25 = 2.34375倍
```

Country Preferredには、戦術別の `tactic_*_preferred_weight_factor` も加算される。概念式は：

```text
country preferred倍率
= 1 + 0.25 + tactic_specific_preferred_weight_factor
```

このMODのArmy Spiritには、`elastic_defense`、`overwhelming_fire`、`infantry_charge`、`delay`、`human_wave_tactics`、`unexpected_thrust`、`breakthrough`、`barrage`、`planned_attack`、`relentless_assault` に `preferred_weight_factor = 1` を与えるものがある。該当SpiritとCountry Preferredが重なれば、その部分は `×2.25` になる。

Ghidraでは `FUN_14126a1e0` 内で三つを別々に乗算している。対応globalは：

- Country: `DAT_1432469f0`
- Army General: `DAT_143246a88`
- Field Marshal: `DAT_143246b18`

指揮官本人がPreferred Tacticを設定できる必要levelは、このMODでは5である。

```lua
PREFERRED_TACTIC_CHARACTER_SKILL_LEVEL_REQUIRED = 5
```

### 1.4 recon・指揮官skill・counter

両軍の戦術選択scoreは：

```text
S = leader skill + (最大reconが敵より高ければ5)
```

低score側が先に戦術を引き、高score側が後から引く。後手側では、先手戦術の `countered_by` に指定された候補だけが：

```text
counter weight倍率 = 1 + 0.35 × |S_A - S_B|
```

になる。実コードは `FUN_141269900` (`0x141269900`) から相手戦術のcounter IDとscore差を渡し、`FUN_14126a1e0` が該当候補だけを増量し、`FUN_141268ca0` (`0x141268ca0`) がweighted randomを行う。

counterが実際に成立すると、counterされた側の戦術modifierは無効化され、counter側の戦術modifierは残る。ただしcounter候補を引くこと自体は保証されない。

詳細なrecon式は別紙 `hoi4_820039020_land_recon_tactics_analysis.md` を参照。

## 2. 戦術データの読み方

| field | 意味 |
|---|---|
| `attacker` | attacker側のdamage dealing factor。`0.25`なら+25% |
| `defender` | defender側のdamage dealing factor。`-0.15`なら-15% |
| `attacker_movement_speed` | attackerの戦闘進行・移動速度補正 |
| `combat_width` | battle全体の使用可能combat width補正 |
| `attacker_org_damage_modifier` | attackerが与えるORG damageへの追加補正 |
| `defender_org_damage_modifier` | defenderが与えるORG damageへの追加補正 |
| rootの`phase` | 選択後にBattle Phaseを変更 |
| trigger内の`phase` | そのphaseでだけ候補になる条件 |
| `countered_by` | 相手側でcounter候補としてweight増量する戦術 |

`attacker` / `defender` はDefenseやBreakthroughの生statを加減するfieldではなく、その陣営の戦闘出力側modifierである。結果として攻撃回数・期待damageを増減させる。

表では `A` をattacker、`D` をdefenderと略す。また、一覧表では読みやすさのためtactic ID先頭の `tactic_` を省略する。たとえば `basic_attack` は `tactic_basic_attack` を指す。

## 3. Battle Phase

通常phaseはデータ上 `no` である。特殊phaseへ入ると、次回の戦術候補がそのphase専用セットへ置き換わる。

```text
通常(no)
 ├─ Assault ───────────────> Close Combat
 │                            └─ CC Withdraw ──> 通常
 │
 ├─ Tactical Withdrawal ───> Tactical Withdrawal
 │                            └─ TW Intercept ─> 通常
 │
 ├─ Seize Bridge ──────────> Seize Bridge
 │                            └─ Retake Bridge ─> Hold Bridge
 │
 ├─ Hold Bridge ───────────> Hold Bridge
 │                            └─ HB Storm ──────> Seize Bridge
 │
 └─ Urban Defense ─────────> Street Fighting
                              └─ 明示的な通常phase復帰戦術なし
```

phaseは師団単位ではなくbattle全体の状態である。Street Fightingは、このMOD定義内に `phase = no` の終了戦術がないため、一度入るとその戦闘中は基本的にStreet Fighting候補を引き続ける。

## 4. counter関係一覧

左側の戦術を相手が先に選んだ場合、右側が後手のcounter候補になる。

| counterされる戦術 | counter候補 |
|---|---|
| `tactic_basic_attack` | `tactic_counterattack` |
| `tactic_assault` | `tactic_counterattack` |
| `tactic_encirclement` | `tactic_tactical_withdrawal` |
| `tactic_delay` | `tactic_shock` |
| `tactic_shock` | `tactic_ambush` |
| `tactic_ambush` | `tactic_breakthrough` |
| `tactic_breakthrough` | `tactic_backhand_blow` |
| `tactic_blitz` | `tactic_elastic_defense` |
| `tactic_masterful_blitz` | `tactic_elastic_defense` |
| `tactic_tw_defend` | `tactic_tw_intercept` |
| `tactic_tw_evade` | `tactic_tw_chase` |
| `tactic_defender_sb_retake_bridge` | `tactic_attacker_sb_skillful_defence` |
| `tactic_attacker_hb_storm` | `tactic_defender_hb_skillful_defence` |
| `tactic_banzai_charge` | `tactic_overwhelming_fire` |
| `tactic_sf_fortify` | `tactic_sf_mouse_holing` |

`Delay → Shock → Ambush → Breakthrough → Backhand Blow` は連鎖状の相性になっている。ただし一回の抽選で両軍が一つずつしか選ばないため、同じ12時間tick内に連鎖が何段も続くわけではない。

## 5. 通常phaseの戦術

### 5.1 初期・一般戦術

| 戦術 | 側 | W | 主な条件 | 効果 | 備考 |
|---|:---:|---:|---|---|---|
| `basic_attack` | A | 4 | 通常phase | A damage +5% | Counterattackでcounter |
| `basic_defend` | D | 4 | 通常phase | D damage +5% | fallback |
| `counterattack` | D | 4 | skill advantage >0 | D damage +25% | `unyielding_defender`でW+4 |
| `assault` | A | 2 | artillery ratio >10% | A damage +25% | Close Combatへ。非UrbanでW×0.2、`aggressive_assaulter`でW+2 |
| `encirclement` | A | 4 | frontage満杯、reserveあり、skill優勢または指定trait | width +50%、A +25%、D +5% | 指定expert traitでW+4 |
| `shock` | A | 4 | 通常phase | D damage -25% | `aggressive_assaulter`でW+4 |
| `ambush` | D | 4 | skill advantage >1、skill>2、または`trickster` | A damage -25% | Breakthroughでcounter |
| `banzai_charge` | A | 4 | JAP | A move +10%、A +25%、D +10% | `active=yes`、Overwhelming Fireでcounter |
| `urban_defense` | D | 2 | Urban | A -5%、D +5%、A move -5%、A ORG damage -10% | Street Fightingへ |

Urban DefenseのweightはVP>5で×2、`urban_assault_specialist`で×3、`trait_engineer`で×1.5。条件が重なれば乗算される。

### 5.2 Doctrine routeなどから候補へ追加される通常戦術

| 戦術 | 側 | W | 主な条件 | 効果 | Counter / Phase |
|---|:---:|---:|---|---|---|
| `delay` | D | 4 | 追加条件なし | A move -25%、A -25%、D -15% | Shockでcounter |
| `tactical_withdrawal` | D | 1 | skill優勢または`trickster` | A -50%、D -5% | Tactical Withdrawal phaseへ |
| `breakthrough` | A | 4 | hardness>50%またはskill advantage>1 | A move +50%、A +25%、D -15% | Backhand Blowでcounter |
| `blitz` | A | 4 | hardness>50%、さらにskill>2 / `panzer_leader` / skill advantage>1 | A move +50%、A +15%、D -15% | Elastic Defenseでcounter。expert traitでW+4 |
| `elastic_defense` | D | 4 | `defensive_doctrine`またはskill>2 | A move -25%、A -15%、D +10% | Blitz系counter候補 |
| `backhand_blow` | D | 4 | skill>4、または`defensive_doctrine`かつskill>3 | A move -30%、A -20%、D +25% | Breakthrough counter候補 |
| `guerrilla_tactics` | D | 4 |指定地形、skill>2または`trickster` | width -50%、A -70%、D -60% | Jungle/UrbanでW×2 |
| `human_wave_tactics` | A | 4 | 指定地形、frontage満杯、reserveあり | width +50%、A +10%、D +10% | Jungle/UrbanでW×2 |
| `infantry_charge` | A | 4 | 追加条件なし | A +10%、D -5% | なし |
| `planned_attack` | A | 4 | 追加条件なし | A +15% | なし |
| `relentless_assault` | A | 4 | 追加条件なし | A move +15%、A +20%、D +5% | なし |
| `unexpected_thrust` | A | 4 | 追加条件なし | A move +100%、A +15%、D -20% | なし |
| `overwhelming_fire` | D | 2 | 追加条件なし | A -10%、D +10% | Banzai counter候補 |
| `barrage` | A | 4 | 追加条件なし | A +10%、D -20% | なし |
| `masterful_blitz` | A | 4 | SOV向け表示、Blitz同系条件 | A move +50%、A +20%、D -20% | Elastic Defenseでcounter。MOD内に候補化する経路なし |

### 5.3 riverから特殊phaseへ入る戦術

| 戦術 | 側 | W | 条件 | 効果 | 移行先 |
|---|:---:|---:|---|---|---|
| `seize_bridge` | A | 2 | river crossing、skill>3または`offensive_doctrine`かつskill>2 | A move +10%、A +20%、D -5% | Seize Bridge |
| `hold_bridge` | D | 2 | river crossing、skill>2または`defensive_doctrine` | A move +10%、A +20%、D -5% | Hold Bridge |

`hold_bridge` はdefender戦術だが、このMODの数値はA moveとA damageを増やし、D damageを減らす。名称から期待する効果と逆に見えるが、上表は実ファイルの値そのままである。

## 6. Close Combat phase

| 戦術 | 側 | W | 効果 | Phase |
|---|:---:|---:|---|---|
| `cc_attack` | A | 4 | A +10%、D +5% | 維持 |
| `cc_defend` | D | 4 | A +5%、D +10% | 維持 |
| `cc_storm` | A | 2 | A +20%、D +20% | 維持 |
| `cc_local_strong_point` | D | 2 | A -20% | 維持 |
| `cc_withdraw` | A | 1 | A -5%、D -5% | 通常へ復帰 |

Close CombatはAssaultから入る。Attackerがweight 1の`cc_withdraw`を引くまで継続する。

## 7. Tactical Withdrawal phase

| 戦術 | 側 | W | 効果 | Counter / Phase |
|---|:---:|---:|---|---|
| `tw_attack` | A | 4 | A -25%、D -35% | 維持 |
| `tw_defend` | D | 4 | A -55%、D -5% | TW Interceptでcounter |
| `tw_chase` | A | 4 | A -15%、D -30% | TW Evade counter候補 |
| `tw_evade` | D | 4 | A -65%、D -10% | TW Chaseでcounter |
| `tw_intercept` | A | 4 | A -5%、D -35% | 通常へ復帰 |

全候補で両軍のdamage outputが低下する。特にdefender側の`tw_defend` / `tw_evade`はattacker出力を大きく下げるため、Tactical Withdrawalは防御側が時間を稼ぐphaseである。

## 8. Bridge phase

### 8.1 Seize Bridge

| 戦術 | 側 | W | 条件 | 効果 | Counter / Phase |
|---|:---:|---:|---|---|---|
| `attacker_sb_hold` | A | 4 | なし | A +20% | 維持 |
| `attacker_sb_skillful_defence` | A | 4 | skill>4 | A +20%、D -10% | Retake Bridge counter候補 |
| `defender_sb_assault` | D | 4 | なし | D -5% | 維持 |
| `defender_sb_reckless_assault` | D | 4 | skill<3 | A +25%、D -10% | 維持 |
| `defender_sb_retake_bridge` | D | 4 | skill>2または`trickster` | A +10%、D -5% | Skillful Defenceでcounter、Hold Bridgeへ |

### 8.2 Hold Bridge

| 戦術 | 側 | W | 条件 | 効果 | Counter / Phase |
|---|:---:|---:|---|---|---|
| `attacker_hb_attack` | A | 4 | なし | A +10% | 維持 |
| `attacker_hb_rush` | A | 4 | skill>4 | A +20% | 維持 |
| `attacker_hb_storm` | A | 2 | なし | A +20%、D +5% | Skillful Defenceでcounter、Seize Bridgeへ |
| `defender_hb_hold` | D | 2 | skill<3 | A +20%、D -10% | 維持 |
| `defender_hb_skillful_defence` | D | 2 | skill>2または`trickster` | A +10%、D +5% | HB Storm counter候補 |

橋梁phaseではdefenderの低skill戦術がattackerを強化する設定が複数あり、高skill・`trickster`条件を満たす価値が大きい。

## 9. Street Fighting phase

| 戦術 | 側 | W | 条件 | 効果 | Counter |
|---|:---:|---:|---|---|---|
| `sf_storm` | A | 2 | なし | A +5%、D +10%、D ORG damage -5% | なし |
| `sf_barrage` | A | 2 | artillery ratio>10% | A +10%、D -5%、D ORG damage -10% | `active=no`、MOD内unlock元なし |
| `sf_armor_supported_assault` | A | 2 | Flame Tank系unitあり | A +15%、D ORG damage -10% | なし |
| `sf_mouse_holing` | A | 2 | EngineerまたはPioneerあり | D -10%、A move -5%、D ORG damage -10% | Fortify counter候補 |
| `sf_defense` | D | 2 | なし | A -5%、D +10%、A ORG damage -10% | なし |
| `sf_fortify` | D | 2 | なし | A -10%、D +5%、A ORG damage -15% | Mouse Holingでcounter |
| `sf_ambush` | D | 2 | なし | D +25%、A ORG damage -10% | `active=no`、MOD内unlock元なし |

Street Fightingでは、通常damage補正に加えて片側のORG damageだけを下げる戦術が多い。Engineer/PioneerやFlame Tankは、ここでは師団stat以外に使用可能戦術そのものを増やす。

## 10. Doctrine routeと国固有戦術

### 10.1 Doctrineから戦術poolへ追加されるもの

同じ戦術が複数のDoctrine routeから候補へ追加される場合がある。`on_research_complete` の `unlock_tactic` と技術直下の `enable_tactic` は同じ技術内で併記されている。

| 戦術 | このMODで確認したDoctrine技術 |
|---|---|
| `unexpected_thrust` | `mobile_warfare` |
| `delay` | `delay`, `sup_delay` |
| `elastic_defense` | `elastic_defence`, `mobile_defence`, `grand_mechanized_offensive`, `armored_operations`, `operational_concentration` |
| `blitz` | `mechanised_offensive`, `armored_spearhead`, `concentrated_fire_plans`, `grand_mechanized_offensive`, `armored_operations`, `deep_operations` |
| `breakthrough` | `blitzkrieg`, `combined_arms`, `shock_and_awe`, `assault_breakthrough`, `breakthrough_priority` |
| `overwhelming_fire` | `kampfgruppe`, `centralized_fire_control`, `overwhelming_firepower`, `assault_concentration`, `vast_offensives` |
| `backhand_blow` | `backhand_blow`, `air_land_battle`, `continuous_offensive` |
| `guerrilla_tactics` | `werwolf_guerillas`, `guerilla_warfare` |
| `barrage` | `superior_firepower` |
| `tactical_withdrawal` | `tactical_control`, `forward_observers` |
| `planned_attack` | `grand_assault` |
| `infantry_charge` | `infantry_offensive` |
| `relentless_assault` | `large_front_operations` |
| `human_wave_tactics` | `human_wave_offensive` |

### 10.2 国固有・削除された戦術

- `banzai_charge` は削除されていない。`tag = JAP`かつ`active = yes`なので、Doctrineを問わず日本だけが使用できる、このMODで唯一実際に使用可能な国tag限定戦術である。
- 現行vanillaにある日本の上位版`grand_banzai_charge`は、このMODの`combat_tactics.txt`から削除されている。vanillaの通常版にある「上位版取得後は通常版を候補外にする」条件もMODでは削除されているため、日本は常に通常の`banzai_charge`を使う。
- `masterful_blitz`の定義と`only_show_for = SOV`は残っているが、`active = no`のままで、使用可能にするeffectがMOD内にない。したがってSOV固有戦術として表示用データは残るが、通常プレイでは抽選候補へ入らない。
- `sf_barrage`と`sf_ambush`も`active = no`のまま候補化する経路がない。

このMODは`replace_path = "common/national_focus"`を指定し、Japan/Sovietのvanilla Focus Treeを収録していない。そのためvanilla Focusが持つ`grand_banzai_charge`や`masterful_blitz`の`unlock_tactic`は流用されない。別MODやconsole/script effectで明示的に登録しない限り、これらは使用不能である。

`common/on_actions/00_ib_strat_up_on_actions .txt` にある多数の `unlock_tactic` は、対応Doctrineを取得済みのsaveへ戦術登録を復元する移行処理であり、独立したDoctrine routeではない。

## 11. 実用上の評価

### 11.1 戦術を増やすことは常に強化ではない

抽選はweight比例なので、Doctrineによって弱い戦術がpoolへ追加されると、強い戦術の当選率を薄める場合がある。反対に、強いcounter・phase移行戦術・編成に合う戦術が追加されるなら価値が高い。

### 11.2 Preferred Tacticの価値

Preferred Tacticは確定ではないが、Generalだけでもweight `×1.5`。Country・General・Field Marshalが同じ戦術を選べば単純条件で `×2.34375` になり、候補数が多い通常phaseほど効きやすい。

### 11.3 reconの価値

reconは直接damageを増やさず、後手を取りcounter候補のweightを上げる。したがって：

- counter可能な強戦術が相手poolにある
- 自軍のDoctrine routeなどでそのcounterが候補化され、triggerも満たす
- 指揮官skill差をreconの+5で逆転または拡大できる

という三条件が揃うほど強い。

### 11.4 特に影響が大きい戦術

- `guerrilla_tactics`: width -50%、A -70%、D -60%。戦闘全体の性質を大幅に変える。
- `encirclement` / `human_wave_tactics`: width +50%。同時に前線へ出せる戦力が変わる。
- `unexpected_thrust`: A movement +100%、D damage -20%。突破速度への影響が大きい。
- `breakthrough` / `blitz`: A movement +50%と敵damage低下を同時に持つ。
- `tactical_withdrawal`: 専用phaseへ移し、数tickにわたり両軍damageを大きく低下させ得る。
- `urban_defense`: Street Fightingへ固定し、以後の戦術poolをUrban専用へ変える。

## 12. Ghidra上の主要関数・global

| 内容 | 関数・global |
|---|---|
| 12時間ごとの更新入口 | `FUN_141265af0` (`0x141265af0`) |
| 両軍score計算・選択順・phase更新 | `FUN_141269900` (`0x141269900`) |
| 候補条件とweight構築 | `FUN_14126a1e0` (`0x14126a1e0`) |
| weighted randomで戦術確定 | `FUN_141268ca0` (`0x141268ca0`) |
| recon最大値集約 | `FUN_141380ff0` (`0x141380ff0`) |
| `TACTIC_SWAP_FREQUENCEY` | `DAT_14324682c` |
| `RECON_SKILL_IMPACT` | `DAT_14324ab40` |
| `INITIATIVE_PICK_COUNTER_ADVANTAGE_FACTOR` | `DAT_143246c38` |
| Country preferred factor | `DAT_1432469f0` |
| General preferred factor | `DAT_143246a88` |
| Field Marshal preferred factor | `DAT_143246b18` |
| Preferred設定必要skill | `DAT_143246940` |

## 13. 確度と注意

### Ghidra実コードとMODデータから確認済み

- 12時間ごとの再抽選
- 候補triggerとscripted base weightの評価
- weighted randomであり、最大weightの確定選択ではないこと
- leader skillとreconによる先手・後手決定
- `countered_by` IDを後手抽選へ渡す経路
- counter候補weightの `1 + 0.35 × score差`
- Country・General・Field Marshal Preferred Tactic補正の別々の乗算
- phase変更fieldを選択後にbattleへ保存する経路
- MODの53戦術、各条件・weight・effect・counter・phase
- Doctrine技術から戦術poolへ追加される対応関係

### 注意

- `masterful_blitz`、`sf_barrage`、`sf_ambush`は、このMOD単体の通常プレイでは候補化する経路がない。別MOD・console・script effectで明示的に登録された場合は別である。
- 数値はこのローカル `hoi4.exe` とMOD 13.00 / 1.17.5.2の組み合わせに対するもの。
- 表のdamage値は戦術modifier単体であり、地形、経験値、planning、entrenchment、air support、装甲・貫徹など他の補正とは別に処理される。
