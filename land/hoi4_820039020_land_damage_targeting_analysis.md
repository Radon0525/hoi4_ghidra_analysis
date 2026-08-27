# HOI4「innovative balance」陸戦ダメージ・攻撃対象選択の内部計算調査

調査対象：Workshop ID `820039020`、MODバージョン `13.00`、対応表記 `1.17.5.2`  
Wiki資料：`Land battle` 改訂版 `102327`、分類 `1.15`  
実行ファイル：ローカルの `hoi4.exe` を Ghidra で静的解析  
調査日：2026-08-25（Defense共有処理・増援確率の追補：2026-08-26）

## 結論

複数師団の攻撃について先に結論を述べると、**Attack 100の師団が3個いる場合、エンジンは3個のAttackを先に300へ合算してから一度だけDefenseと比較するわけではない**。各攻撃師団について、割り当てられた攻撃対象と攻撃量を個別に作り、命中判定関数を別々に呼ぶ。一方で、攻撃を受ける師団が持つ「その戦闘時間中に使用済みのDefense/Breakthrough回数」のカウンタ（combat participantオブジェクトの `+0x344`）は、複数の攻撃元から共通して参照・加算される。したがって正確な表現は、**「Attackは100を3回別処理するが、同じ対象を攻撃する限り、その3回は対象側の一本のDefense枠を順番に消費する」**である。

表示上のSoft Attack 100は、通常は内部の攻撃回数へ変換すると約10 pipsである。たとえばHardness 0%、3師団とも同じ対象へ100%を割り当て、各師団のSoft Attackが100、対象のDefenseが150で、他の補正や丸め差を無視すると、攻撃側は10 pipsを3回、対象側は合計15 Defense pipsを共有する。処理順に見ると第1師団の10 pips、第2師団の最初の5 pipsまではdefended、残り15 pipsはundefendedとなり、期待命中数は `15 × 10% + 15 × 40% = 7.5` である。この限定条件では「Attack 300対Defense 150」とまとめた期待値計算と一致するが、コード上の処理が一回に合算されていることを意味しない。target allocation、師団ごとの補正、Soft/Hard比、stochastic rounding、処理順、各命中後のdamage乱数が異なるため、一般には個別処理として扱う必要がある。

戦闘中のreserveからfrontへ入る増援判定については、**reserve師団全員がそれぞれ独立にreinforce chanceを振る方式ではない**。一時間ごと・戦闘陣営ごとに、参加可能なreserve師団から一個をweighted randomで選び、選ばれた一個についてreinforce chanceを一回だけ判定する。したがって、同じreinforce chanceを持つreserveが十個あっても、「その時間に誰か一個が入る確率」は十倍にならない。このMODの基礎値2%だけなら、誰か一個が入る確率は2%、特定の一個が入る確率は0.2%である。詳しくは第18節で説明する。

陸戦の1時間分の処理は、概念的には次の順序になっている。

```text
各攻撃師団
  1. 敵師団の順序を無作為に並べ替える
  2. 交戦可能幅に収まる攻撃対象を抽出する
  3. 集中攻撃の配分率 C を計算する
  4. 各対象へ分散攻撃分を戦闘正面幅の比率で配る
  5. 優先度評価値が最大の対象を選ぶ
  6. 優先対象へ集中攻撃分 C を追加で渡す

攻撃師団・対象師団・攻撃配分の各組み合わせ
  7. Soft Attack / Hard Attack と対象の Hardness から有効攻撃回数を作る
  8. 攻撃配分と各種戦闘補正を掛け、確率的丸めで攻撃回数を整数化する
  9. 対象の Defense または Breakthrough を消費しながら、各攻撃の命中を判定する
 10. 命中ごとに STR 用ダイスと ORG 用ダイスを別々に振る
 11. 戦力の10%段階値、戦闘ダメージ補正、貫徹段階を適用する
 12. 対象の HP / Organization へダメージを確定する
```

Wikiの説明は処理の大枠としてはかなり正しい。ただし、このMODの実値と実コードには重要な差がある。

| 項目 | 添付Wiki | このMOD / 実コード |
|---|---:|---:|
| 集中攻撃の基礎配分 | 35% | **30%** |
| 集中攻撃配分の上限 | 90% | **90%** |
| 通常時のSTRダイス | d2 | **d2** |
| 通常時のORGダイス | d4 | **d4** |
| 装甲優位時のSTRダイス | d2 | **d2** |
| 装甲優位時のORGダイス | d6 | **d5** |
| STRダメージ補正 | 0.06 | **0.05** |
| ORGダメージ補正 | 0.053 | **0.05** |
| 防御値が残っている間の命中率 | 10% | **10%** |
| 防御値を使い切った後の命中率 | 40% | **40%** |

さらに、Wikiの `round()` は通常の四捨五入ではない。攻撃回数と防御回数の小数部分は、RNG（乱数生成器）を使う **確率的丸め（stochastic rounding）** で処理される。詳しくは第5節で説明する。

## 1. 攻撃対象リストの作成

主要関数は `FUN_141265c00` (`0x141265c00`) で、攻撃対象リストの補助関数は `FUN_1412623f0` (`0x1412623f0`) である。

各攻撃師団について、敵師団の配列を Fisher–Yates 型の処理で無作為に並べ替える。その後、次の交戦可能幅を作る。

```text
engagement_width
= attacker_combat_width × ENGAGEMENT_WIDTH_PER_WIDTH
```

このMODでは次の値である。

```lua
ENGAGEMENT_WIDTH_PER_WIDTH = 2.0
```

したがって、通常は自身の戦闘正面幅の2倍までを攻撃対象リストに入れられる。

並べ替え済みの敵師団リストを先頭から最後まで走査し、次を満たす師団だけを追加する。

```text
selected_width + candidate_width <= engagement_width
```

入らない候補は除外するが、その時点で探索を止めず、後続のより幅が狭い師団が入るかを引き続き調べる。ちょうど上限に達した場合だけ、その場で終了する。

一つも入らなかった場合は、並べ替え後の先頭にいる敵師団を強制的に一つ追加する。この代替処理では交戦可能幅の超過を許す。したがって「どの敵師団も幅が広すぎる場合は一つを無作為に選ぶ」というWikiの説明と一致する。

### 1.1 攻撃対象リストに関する実用上の意味

- 攻撃対象リストは各攻撃師団ごとに別々に作られる。
- 幅の広い攻撃師団ほど、一度に候補にできる敵師団の合計幅も大きい。
- 幅が広すぎて入らない敵師団が途中にいても、後続の幅が狭い敵師団は候補になり得る。
- リストの順序は無作為なので、同じ編成同士でも対象集合は毎時変わり得る。
- 優先度評価値が完全に同点なら、並べ替え後に先に現れた対象が残る。

## 2. 集中攻撃と分散攻撃への配分

実コードの集中攻撃配分を `C` とすると、正規化した式は次のとおり。

```text
C = min(
      0.90,
      DAMAGE_SPLIT_ON_FIRST_TARGET
      + coordination_bonus × (1 + attacker_initiative)
    )
```

このMODでは：

```lua
DAMAGE_SPLIT_ON_FIRST_TARGET = 0.30
```

よって基礎値は **30%** であり、添付Wikiの35%ではない。実コードは上限 `0.90` を即値で持ち、defineには公開していない。これはMOD側コメントとも一致する。

実行時には `coordination_bonus` に対応する補正値を読み、攻撃師団の陸軍統計値ブロックにある `Initiative` を `1 + Initiative` として掛けている。

この箇所には上限 `min(0.90, ...)` はあるが、明示的な下限 `max(0, ...)` はない。通常のゲームデータ範囲では問題にならないが、極端な負の `coordination_bonus` をMODで作る場合は注意が必要である。

分散攻撃の配分 `U` は：

```text
U = 1 - C
```

選ばれた各対象 `i` へ渡す配分は：

```text
uncoordinated_share_i
= target_width_i / sum(selected_target_width) × U
```

全計算は `100000 = 1.0` の固定小数点で、各整数除算の段階で切り捨てが入る。そのため対象数が多い場合、配布率の合計は理論値 `U` よりごくわずかに小さくなることがある。

優先対象には、上記の戦闘正面幅比による配分に加え、`C` 全体がもう一度渡される。

重要なのは、優先対象に対する配分を先に合算して一回だけダメージ計算するのではない点である。コードは：

1. 分散攻撃の処理を選ばれた全対象へ順番に実行
2. その後、優先対象へ集中攻撃の処理をもう一回実行

という二回の独立した呼び出しを行う。このため攻撃回数の確率的丸めも二回に分かれる。

## 3. 優先攻撃対象の評価値

選ばれた対象が一つだけなら、その師団が無条件で優先対象になる。二つ以上の場合、各対象について次の評価値を計算する。

```text
base_score
= (SoftAttack / 10) × (1 - target_hardness) × SOFT_ATTACK_TARGETING_FACTOR
 + (HardAttack / 10) × target_hardness       × HARD_ATTACK_TARGETING_FACTOR

armor_factor
= 0.5  if attacker_piercing < target_armor
   1.0  otherwise

org_factor
= 1 - target_org_ratio / 4

priority_score
= base_score × armor_factor × org_factor
```

このMODでは：

```lua
SOFT_ATTACK_TARGETING_FACTOR = 1.0
HARD_ATTACK_TARGETING_FACTOR = 1.2
```

つまり攻撃対象選択のときだけ Hard Attack 側を20%高く評価する。この `1.2` は実ダメージのSoft/Hard合成には使わない。あくまで優先対象選択の重みである。

### 3.1 Armorによる減点は二択判定

優先度評価値のArmor判定は、段階的なPiercing表を使わない。

```text
attacker_piercing < target_armor なら score を半分
```

したがってPiercing比率が99%でも優先度評価値は `×0.5`、100%以上なら `×1.0` である。一方、実ダメージには後述の4段階のPiercing倍率を使う。この二つを混同してはいけない。

### 3.2 低ORGの対象を優先する効果はそれほど大きくない

`org_ratio` が100%なら `org_factor = 0.75`、0%なら `1.00` である。したがって同じ攻撃適性なら、ORGゼロ近辺の対象はORG満タンの対象の最大 `1 / 0.75 = 1.333...` 倍の評価値になる。

### 3.3 同点時の選択

比較は「新しい評価値が現在の最高値より大きい場合だけ更新」である。同点では先に選ばれた対象を維持するため、完全同点の最終結果は事前の無作為な並べ替え順に依存する。

## 4. ダメージ計算の入口

攻撃師団・対象師団・攻撃配分の各組み合わせは、`FUN_141260a30` (`0x141260a30`) へ渡される。

対象のHardnessを `H` としたSoft/Hard合成の中心は：

```text
hardness_modified_attack
= (SoftAttack / 10) × (1 - H)
 + (HardAttack / 10) × H
```

この値に攻撃師団の現在の戦闘攻撃補正、状況別補正、攻撃対象選択処理から渡された配分を順に掛ける。

```text
expected_attack_pips
= hardness_modified_attack
 × attacker_combat_attack_factor
 × situational_attack_factors
 × target_share
```

途中は固定小数点の整数演算なので、各乗算の後に小さな切り捨てが発生する。

## 5. `round(attack / 10)` の正体

`expected_attack_pips` は普通の四捨五入ではなく、整数部を確定させ、小数部分だけをその確率で1へ繰り上げる。この方式を **確率的丸め（stochastic rounding）** という。

```text
attack_pips
= floor(expected_attack_pips)
 + Bernoulli(frac(expected_attack_pips))
```

例：

| 計算結果 | 実際の整数化 |
|---:|---|
| 10.00 | 常に10 |
| 10.24 | 76%で10、24%で11 |
| 10.50 | 50%で10、50%で11 |
| 10.99 | 1%で10、99%で11 |

実装は `0..99` の100段階乱数を使うため、小数部分も実質1%刻みで判定される。

これは期待値を保つので長期平均はWiki式に近いが、単一の時間処理では通常の四捨五入と違う結果になる。

### 5.1 RNGとは何か

`RNG` は **Random Number Generator** の略で、日本語では「乱数生成器」または「乱数生成処理」という。ゲームは本物のサイコロを振るのではなく、初期値と現在の内部状態から、乱数に見える整数列を順番に作る。

HOI4のこの処理では、たとえば `0～99` のどれかを一つ生成し、命中、丸め、ダイス目などの判定に使う。一般的なゲームのRNGと同じく疑似乱数なので、同じ初期状態と同じ呼び出し順序を完全に再現できれば、同じ結果列を再現できる。

### 5.2 なぜ確率的丸めを使うのか

普通の四捨五入では `10.24` は常に10になり、小さい端数が毎時間失われる。確率的丸めなら24%で11になるため、多数回繰り返した平均値は10.24へ近づく。

```text
10.24を100時間処理する例

普通の四捨五入：おおむね 10 × 100 = 1000
確率的丸め：      おおむね 10.24 × 100 = 1024
```

つまり、各時間の結果にはばらつきを持たせながら、長期平均では小数部分を失わないための丸め方である。

## 6. Defense / Breakthrough と命中率

命中判定の補助関数は `FUN_140c0acb0` (`0x140c0acb0`) である。

攻撃される師団が戦闘の防御側なら `Defense`、攻撃側なら `Breakthrough` に対応する集計値を選び、概ね次を作る。

```text
expected_defense_pips
= (Defense_or_Breakthrough / 10) × current_defense_factor
```

これも小数部分を確率的丸めする。対象師団の内部データには、その時間ですでに消費した防御回数のカウンターがあり、各攻撃について：

```text
if used_defense < defense_pips:
    used_defense += 1
    hit chance = 1 - BASE_CHANCE_TO_AVOID_HIT
else:
    hit chance = 1 - CHANCE_TO_AVOID_HIT_AT_NO_DEF
```

このMODでは：

```lua
BASE_CHANCE_TO_AVOID_HIT       = 90
CHANCE_TO_AVOID_HIT_AT_NO_DEF = 60
```

したがって：

| 状態 | 回避率 | 命中率 |
|---|---:|---:|
| 防御回数が残っている | 90% | **10%** |
| 防御回数を使い切った | 60% | **40%** |

### 6.1 防御回数は攻撃師団ごとではなく、攻撃される師団に属する

消費済みカウンターは攻撃される師団の内部データに置かれている。したがって、複数の敵師団から同じ対象へ届いた攻撃、同じ攻撃師団の分散攻撃、優先対象への集中攻撃は、同じ時間帯の残り防御回数を順に消費する。

攻撃対象選択関数の実行順は「選ばれた各対象への分散攻撃」→「優先対象への集中攻撃」であるため、同じ時間でも処理順は結果に影響し得る。

### 6.2 Ghidraで確認した「攻撃は別処理、Defenseは共有」の実装

この点は、単に合計値から推測したものではなく、次の三段階をGhidra上で確認した。

#### A. 攻撃師団は外側loopで一個ずつ処理される

`FUN_141265c00` (`0x141265c00`) では、攻撃側の参加師団vectorを外側loopで走査する。各攻撃師団について対象listを作り直し、各対象への分散攻撃shareごとに `FUN_141260a30` (`0x141260a30`) を呼び、最後に優先対象への集中攻撃shareでも同じ関数をもう一度呼ぶ。

概念的には次の構造である。

```text
for attacker in attacking_divisions:
    targets = build_target_list(attacker)

    for target in targets:
        resolve_attacker_target_share(
            attacker,
            target,
            uncoordinated_share_for_target
        )

    resolve_attacker_target_share(
        attacker,
        preferred_target,
        coordinated_share
    )
```

したがって、Attack 100の師団が三個いても、最初に `100 + 100 + 100 = 300` という一個のAttack statへ合成する処理はない。攻撃師団・対象・shareの組み合わせごとに、攻撃回数の作成、確率的丸め、命中判定、damage diceまで別々に実行する。

#### B. Defense消費値はtargetオブジェクトの `+0x344`

`FUN_141260a30`は、その組み合わせで発生した整数attack pipsを `FUN_1412675e0` (`0x1412675e0`) へ渡す。後者はattack pipごとのloopで `FUN_140c0acb0` (`0x140c0acb0`) を呼ぶ。

`FUN_140c0acb0`の重要部分は次の形である。

```text
rounded_defense_pips = stochastic_round(
    Defense_or_Breakthrough / 10 × current_defense_factor
)

if rounded_defense_pips > target.used_defense_pips_at_0x344:
    target.used_defense_pips_at_0x344 += 1
    avoid_chance = 90%
else:
    avoid_chance = 60%
```

ここで第一引数は攻撃師団ではなく、**攻撃されているtarget師団のcombat participantオブジェクト**である。異なる攻撃師団から同じtarget pointerを渡せば、全呼び出しが同じ `target + 0x344` を読み書きする。

なお、ゲームは表示Defense stat自体を減算しているわけではない。表示Defenseはそのまま保持し、「この時間にDefenseで保護したattack pips数」を `+0x344` で数える。本文でいう「Defense残量」は、次の概念的な値を指す。

```text
conceptual_remaining_defense
= rounded_defense_pips - used_defense_pips_at_0x344
```

#### C. combat update開始時に参加師団すべてを0へresetする

`FUN_14125ce80` (`0x14125ce80`) はcombat updateの冒頭で `FUN_141380d20` (`0x141380d20`) を呼ぶ。`FUN_141380d20`は参加師団listを全走査し、各combat participantについて次を実行する。

```text
for participant in combat.participants:
    participant.used_defense_pips_at_0x344 = 0
```

その後、その時間の各攻撃師団・各target shareが同じfieldを順番に増やす。このため共有範囲は「戦闘全体で永久」でも「攻撃師団一個ごと」でもなく、**一回のcombat update、すなわちその1時間tick内の対象師団ごと**である。

### 6.3 「Attack 100の師団が三個」の正確な数値例

ここでは表示statと内部attack pipsを混同しないよう、次を仮定する。

- 三個の攻撃師団はすべて表示Soft Attack 100
- targetはHardness 0%で、Hard Attackは影響しない
- 三個とも同じ一個の敵師団へshare 100%を渡す
- strength、地形、tacticなど他のattack補正は100%
- 敵は防御側で表示Defense 150
- 小数部分と確率的丸めは発生しない

各攻撃師団は個別に：

```text
表示Soft Attack 100
→ internal attack pips = 100 / 10
→ 10 pips
```

敵のDefenseは：

```text
表示Defense 150
→ internal defense pips = 150 / 10
→ 15 pips
```

実際の順次処理は次のようになる。

| 処理 | attack pips | 開始時used Defense | 10%命中側 | 40%命中側 | 終了時used Defense |
|---|---:|---:|---:|---:|---:|
| 攻撃師団1 | 10 | 0 | 10 | 0 | 10 |
| 攻撃師団2 | 10 | 10 | 5 | 5 | 15 |
| 攻撃師団3 | 10 | 15 | 0 | 10 | 15 |
| 合計 | 30 | — | 15 | 15 | — |

期待命中数は：

```text
15 × 0.10 + 15 × 0.40
= 1.5 + 6.0
= 7.5 hits
```

したがって質問への最も正確な答えは：

> 表示Attackを300へ合算してから一回だけ処理するのではなく、表示Attack 100を持つ師団を三回別々に処理する。ただし同じtargetへ届いた内部attack pipsは、targetが持つ同じ15 Defense pipsを累積して消費する。

平均値だけを簡略計算する場合には：

```text
合計表示Attack 300 → 30 internal attack pips
表示Defense 150   → 15 internal defense pips
```

としても上の期待値と一致する。しかし、これは**限定条件下の期待値近似**であり、実コードがAttack 300の一括処理をしているという意味ではない。

### 6.4 合算近似が一致する条件と一致しない条件

三師団分を単純合算してよいのは、概ね次がすべて成立するときの期待値計算に限られる。

- 三師団の攻撃が同じtargetへ入る
- 各師団のSoft/Hard Attack構成と各種damage modifierを同一視できる
- target shareをすでに反映済み
- 確率的丸めの分布差を無視する
- どの師団のattack pipsが10%側／40%側へ割り当てられるかを区別しない

実戦では、次の理由で厳密には一致しない。

1. **target shareが師団ごとに違う。** 各師団が異なる対象list・優先対象を持つため、表示Attack 300すべてが一師団へ入るとは限らない。
2. **確率的丸めが別々である。** 例えば各師団から同じtargetへ6.5 pipsずつ届く場合、コードは6.5を三回別々に丸める。19.5を一回だけ丸める処理ではない。
3. **攻撃師団ごとの補正が先に掛かる。** strength、Soft/Hard構成、tactic、地形、air supportなどが違えば、同じ表示Attack 100でもtargetへ届く有効attack pipsは異なる。
4. **damage diceも呼び出しごと・命中ごとに別である。** 合計期待値が同じでも実際のSTR/ORG damage分布は一括固定値にならない。
5. **処理順が10%側と40%側の担当を決める。** 先に処理された攻撃がDefense内を使い、後続の攻撃ほどDefense超過側へ回りやすい。攻撃師団が異質なら、どの師団の攻撃が超過側になるかも結果へ影響し得る。

### 6.5 防御値に小数部分がある場合

コードは一つの `0..99` の乱数を、その攻撃における防御回数の確率的丸めと、10%/40%の命中判定の両方に再利用する。Defense/10 が整数なら単純な10%/40%だが、小数が残る場合は二つの判定が完全には独立でない。

## 7. 命中後のSTR / ORGダイス

ダイス処理は `FUN_1412675e0` (`0x1412675e0`) で行う。命中した攻撃ごとに、STR用とORG用の別々の乱数を生成する。

通常時：

```text
STR roll ~ UniformInteger(1, LAND_COMBAT_STR_DICE_SIZE)
ORG roll ~ UniformInteger(1, LAND_COMBAT_ORG_DICE_SIZE)
```

このMODでは：

```lua
LAND_COMBAT_STR_DICE_SIZE = 2
LAND_COMBAT_ORG_DICE_SIZE = 4
```

よって：

| ダメージ | ダイス目 | 平均値 |
|---|---|---:|
| STR | 1..2 | 1.5 |
| ORG | 1..4 | 2.5 |

## 8. 装甲優位の実装

装甲優位は、「攻撃師団のArmorを対象師団のPiercingが100%貫徹できるか」で判定する。

```text
if target_piercing / attacker_armor < 1.0:
    armor-advantage dice を使用
else:
    normal dice を使用
```

この条件は `PIERCING_THRESHOLDS` の段階番号が0より大きいかで実装されている。85%や70%の部分貫徹でも、100%に達していなければ攻撃師団は装甲優位ダイスを得る。

このMODの装甲優位ダイスは：

```lua
LAND_COMBAT_STR_ARMOR_ON_SOFT_DICE_SIZE = 2
LAND_COMBAT_ORG_ARMOR_ON_SOFT_DICE_SIZE = 5
```

したがって：

| ダメージ | 通常時 | 装甲優位時 | 平均値の変化 |
|---|---:|---:|---:|
| STR | d2、平均1.5 | d2、平均1.5 | 変化なし |
| ORG | d4、平均2.5 | d5、平均3.0 | **+20%** |

添付Wikiの d6、平均3.5、+40% はこのMODには当てはまらない。

## 9. Piercingによる最終ダメージ倍率

装甲優位ダイスとは別に、「攻撃師団のPiercingが対象師団のArmorをどこまで貫徹するか」で最終STR/ORGダメージを減衰させる。

```lua
PIERCING_THRESHOLDS = {
    1.00,
    0.85,
    0.70,
    0.00,
}

PIERCING_THRESHOLD_DAMAGE_VALUES = {
    1.00,
    0.80,
    0.60,
    0.40,
}
```

実コードでは境界値を含む。

| 攻撃師団のPiercing / 対象のArmor | STR/ORGダメージ倍率 |
|---:|---:|
| `>= 1.00` | 1.00 |
| `>= 0.85` かつ `< 1.00` | 0.80 |
| `>= 0.70` かつ `< 0.85` | 0.60 |
| `< 0.70` | 0.40 |

対象のArmorが0の場合は最大ダメージ扱いになる。

ここで二方向のPiercing判定を分ける必要がある。

```text
対象のPiercing 対 攻撃師団のArmor
  -> 攻撃師団が装甲優位ダイスを得るか

攻撃師団のPiercing 対 対象のArmor
  -> 攻撃師団の最終STR/ORGダメージ倍率
```

両者は同じ関数内で別々に計算される。

### 9.1 旧仕様と思われるdeflection defines

MODには次も残っている。

```lua
LAND_COMBAT_STR_ARMOR_DEFLECTION_FACTOR = 0.5
LAND_COMBAT_ORG_ARMOR_DEFLECTION_FACTOR = 0.5
```

しかし今回確認した中核の実ダメージ経路 `FUN_141260a30` は、最終ダメージ軽減にこれらではなく `PIERCING_THRESHOLD_DAMAGE_VALUES` を読む。deflection factorsへの直接参照は主にUIや予想ダメージ表示の関数で確認され、実ダメージ確定経路では使われていない。

少なくともこのビルドで実戦ダメージを再現するなら、`0.5`を一律に掛けるのではなく、上記 `1.0 / 0.8 / 0.6 / 0.4` の表を使うべきである。

## 10. 現在戦力によるダメージ出力補正

ダメージダイスへ渡す攻撃師団の現在戦力係数は、補助関数 `FUN_140c0e8a0` (`0x140c0e8a0`) で10%刻みの段階値へ落とされる。

正規化すると：

```text
strength_bucket
= max(0.10, floor(current_strength_ratio / 0.10) × 0.10)
```

例：

| 現在戦力 | 適用される段階値 |
|---:|---:|
| 100% | 100% |
| 99% | 90% |
| 90% | 90% |
| 89% | 80% |
| 12% | 10% |
| 1% | 10% |

これは「最も近い10%」への丸めではなく、**10%単位の切り捨て、最低10%** である。添付Wikiの「100%未満かつ90%以上なら90%」という例とは一致する。

実行時にはこの段階値を、戦闘陣営側に保存されたSTR/ORGダメージ補正や、その他の事前計算済み補正と組み合わせて各ダイスのダメージへ適用する。

## 11. 最終STR / ORGダメージ

重要部分だけ正規化すると、攻撃師団・対象師団・攻撃配分ごとの最終ダメージは次の形である。

```text
STR_damage
= sum(successful STR die contributions)
 × LAND_COMBAT_STR_DAMAGE_MODIFIER
 × piercing_damage_multiplier

ORG_damage
= sum(successful ORG die contributions)
 × LAND_COMBAT_ORG_DAMAGE_MODIFIER
 × piercing_damage_multiplier
```

このMODでは：

```lua
LAND_COMBAT_STR_DAMAGE_MODIFIER = 0.050
LAND_COMBAT_ORG_DAMAGE_MODIFIER = 0.050
```

添付Wikiの `0.06` と `0.053` をこのMODの再現計算へ使うと過大になる。

ダイスによるダメージには少なくとも次がすでに組み込まれている。

- 1～ダイス最大値の無作為な出目
- 10%刻みの現在戦力段階値
- 攻撃側・防御側の事前計算済みダメージ補正
- 攻撃師団・対象師団固有の戦闘補正
- ORGダメージ側だけに入る追加ORGダメージ係数

その後に全体共通のSTR/ORG補正とPiercing段階を掛け、対象のHP/ORG更新処理へ渡す。

## 12. 一つの数式にまとめる場合の近似形

実コードは整数切り捨て、確率的丸め、攻撃配分ごとの別処理、攻撃ごとの乱数生成を含むため、一つの閉じた式には完全にはできない。期待値だけなら概ね：

```text
E[attack pips to target]
≈ {SA/10 × (1-H) + HA/10 × H}
  × attack modifiers
  × target share

E[hits]
≈ min(attacks, defense_pips) × 0.10
 + max(0, attacks - defense_pips) × 0.40

E[STR damage]
≈ E[hits]
  × mean(STR die)
  × strength/combat damage factors
  × 0.05
  × piercing multiplier

E[ORG damage]
≈ E[hits]
  × mean(ORG die)
  × strength/combat damage factors
  × 0.05
  × piercing multiplier
```

ただし実際のシミュレーションでは、複数の攻撃師団が同じ対象の防御回数を共有し、各攻撃配分が別々に確率的丸めされるため、対象単位に時系列で処理しないと完全一致しない。

### 12.1 添付Wikiの例をこのMOD値で置き換える

次を仮定する。

- Soft Attack 1000、対象のHardness 0%
- 対象のDefense 500
- 攻撃師団の現在戦力段階値90%
- 対象は攻撃師団のArmorを完全貫徹できず、攻撃師団は装甲優位を得る
- 攻撃師団は対象を完全貫徹する
- その他のダメージ補正はすべて増減なし

すると攻撃回数は100、防御回数は50で、期待命中数は：

```text
50 × 0.10 + 50 × 0.40 = 25回命中
```

このMODの装甲優位時ORGダイスはd5なので平均3.0、ORG全体補正は0.05である。

```text
E[ORG damage]
= 25 × 3.0 × 0.90 × 0.05
= 1時間当たり3.375
```

STRダイスはd2、平均1.5、STR全体補正は0.05なので：

```text
E[STR damage]
= 25 × 1.5 × 0.90 × 0.05
= 1時間当たり1.6875
```

装甲優位がなければORGダイス平均は2.5となり、同条件の期待ORGダメージは `2.8125` になる。したがって、このMODにおける装甲優位のORG期待値上昇は `3.375 / 2.8125 = 1.20`、つまり20%である。

## 13. 攻撃対象選択とDefenseの相互作用

Coordinationの価値は総ダメージを直接増やすことではなく、優先対象へ攻撃回数を集中させ、その対象のDefense/Breakthroughを越える確率を上げることにある。

例として有効攻撃回数の合計が100、対象A/Bが同じ戦闘正面幅、`C = 0.30` なら、固定小数点誤差を無視すると：

```text
分散攻撃分 = 70
A = 35
B = 35

優先対象がAなら集中攻撃30を追加
Aの合計 ≈ 65
Bの合計 ≈ 35
```

同じ100回の攻撃でも均等に50/50へ分けるより、AのDefenseを突破して40%命中領域へ入る可能性が高くなる。一方、Aを素早く退場させることで、敵全体のORG/HPを減らす効果もある。

ただし優先対象候補のArmorを完全貫徹できない場合、優先度評価値は半分になるため、ゲームはその対象への集中を避ける傾向がある。これは実ダメージに使う段階的なPiercing軽減とは別の二択式選択ルールである。

## 14. Wiki説明との判定

| Wikiの説明 | 判定 | 補足 |
|---|---|---|
| 交戦可能幅は自身の2倍 | 一致 | MODのdefine値2.0を実行時に読む |
| 敵師団の順序は無作為 | 一致 | 攻撃師団ごとに並べ替える |
| 幅に入らない対象は除外 | 一致 | 後続候補の探索は続く |
| 一つも入らなければ無作為な一対象 | 一致 | 並べ替え後の先頭を強制追加 |
| 分散攻撃分は戦闘正面幅に比例 | 一致 | 固定小数点の切り捨てあり |
| 集中攻撃分は優先対象一体へ | 一致 | 分散攻撃処理後に追加処理 |
| 集中攻撃配分 = 35% + Coordination × (1+Initiative) | **基礎値だけ不一致** | このMODは30%、上限90% |
| Hard Attackは優先度評価で1.2倍 | 一致 | 実ダメージのSoft/Hard合成には使わない |
| 完全貫徹できない対象の優先度は半分 | 一致 | 二択判定 |
| 低ORGを優先 | 一致 | `1 - org_ratio/4` |
| 攻撃・防御回数は`round(x/10)` | 要修正 | 確率的丸め |
| 防御中10%、超過後40%命中 | 一致 | MODのdefine値と実行コードが一致 |
| 通常ダイスはSTR d2 / ORG d4 | 一致 | 実行コードからの参照を確認 |
| 装甲優位時ORGダイスはd6 | **不一致** | このMODはd5 |
| 装甲優位はORG平均+40% | **不一致** | このMODは+20% |
| STR/ORG補正は0.06 / 0.053 | **不一致** | このMODは0.05 / 0.05 |
| 現在戦力による出力は10%刻み | 一致 | 厳密には切り捨て、最低10% |

## 15. 主要関数・グローバル変数対応

| 役割 | アドレス / グローバル変数 |
|---|---|
| 1時間ごとの攻撃対象選択・攻撃配分 | `FUN_141265c00` @ `0x141265c00` |
| 交戦可能幅に基づく対象リスト作成 | `FUN_1412623f0` @ `0x1412623f0` |
| 攻撃師団・対象・配分ごとのダメージ処理入口 | `FUN_141260a30` @ `0x141260a30` |
| 攻撃ごとの命中判定・防御回数消費 | `FUN_140c0acb0` @ `0x140c0acb0` |
| 命中ごとのSTR/ORGダイス | `FUN_1412675e0` @ `0x1412675e0` |
| combat update冒頭の参加師団reset呼び出し | `FUN_14125ce80` @ `0x14125ce80` |
| 全参加師団の消費済みDefense counterを0へreset | `FUN_141380d20` @ `0x141380d20` |
| 対象師団の消費済みDefense counter | combat participant object `+0x344` |
| 現在戦力を10%刻みにする補助関数 | `FUN_140c0e8a0` @ `0x140c0e8a0` |
| `ENGAGEMENT_WIDTH_PER_WIDTH` | `DAT_143246a18` |
| `SOFT_ATTACK_TARGETING_FACTOR` | `DAT_143247f38` |
| `HARD_ATTACK_TARGETING_FACTOR` | `DAT_143247fb0` |
| `DAMAGE_SPLIT_ON_FIRST_TARGET` | `DAT_143249ae8` |
| `BASE_CHANCE_TO_AVOID_HIT` | `DAT_143246640` |
| `CHANCE_TO_AVOID_HIT_AT_NO_DEF` | `DAT_1432466d0` |
| `LAND_COMBAT_STR_DICE_SIZE` | `DAT_1432458d8` |
| `LAND_COMBAT_ORG_DICE_SIZE` | `DAT_143245848` |
| `LAND_COMBAT_STR_ARMOR_ON_SOFT_DICE_SIZE` | `DAT_143245a8c` |
| `LAND_COMBAT_ORG_ARMOR_ON_SOFT_DICE_SIZE` | `DAT_143245b24` |
| `LAND_COMBAT_STR_DAMAGE_MODIFIER` | `DAT_143245cc8` |
| `LAND_COMBAT_ORG_DAMAGE_MODIFIER` | `DAT_143245d70` |
| `PIERCING_THRESHOLDS` | `DAT_14324d130` |
| `PIERCING_THRESHOLD_DAMAGE_VALUES` | `DAT_14324d148` |
| reserveから増援候補を一個選び成功判定する関数 | `FUN_141261340` @ `0x141261340` |
| 増援判定の呼び出しとreserve→active移動 | `FUN_141260340` @ `0x141260340` |
| 師団ごとのreinforce chance計算 | `FUN_140c2bf20` @ `0x140c2bf20` |
| `REINFORCE_CHANCE` | `DAT_143249bf0` |
| `SPEED_REINFORCEMENT_BONUS` | `DAT_143249c88` |
| `ARMY_INITIATIVE_REINFORCE_FACTOR` | `DAT_143247060` |

## 16. 確度

### 実行コードとMODデータから確認済み

- 攻撃対象リストの無作為な順序と、交戦可能幅への採用・除外・代替処理
- 集中攻撃・分散攻撃の処理順
- 基礎配分0.30、上限0.90、`Coordination × (1 + Initiative)` の形
- 戦闘正面幅に比例する分散攻撃配分
- 優先度評価値のSoft/Hard/Armor/ORG各係数
- 攻撃回数と防御回数の確率的丸め
- 防御回数の消費カウンターが対象師団の `+0x344` にあり、複数攻撃師団で共有されること
- combat update冒頭に参加師団すべての `+0x344` が0へresetされること
- Defense内10%／Defense超過40%の命中判定
- 通常時と装甲優位時のダイス最大値
- 装甲優位と最終Piercing軽減が、別方向の二つの判定であること
- 4段階のPiercingダメージ表
- STR/ORG全体ダメージ補正0.05 / 0.05
- 現在戦力を10%単位で切り捨て、最低10%とする処理
- reserve全師団の独立抽選ではなく、候補から一個をweighted randomで選んだ後に一回だけ増援抽選すること
- 同じreinforce chanceのreserve数を増やしても、一時間当たりの最初の増援成功率は上がらないこと
- 増援候補のweightがおおむねreinforce chanceの二乗に比例すること
- 成功した一師団だけがreserve listからactive listへ移ること

### 確度の高いフィールド名の特定

Ghidraには型名・フィールド名が残っていないため、集計済み統計値ブロックの一部はオフセットで読まれる。ただし、Soft Attack、Hard Attack、Hardness、Armor、Piercingは計算式とUI側からの参照で相互確認できる。`Coordination × (1 + field@+0xe0)` のフィールドは、ゲームデータの師団統計値配列順、Wiki式、実際の乗算形から `Initiative` と判定できる。

## 17. 再現実装時の注意

完全一致を狙うなら、次を省略してはいけない。

1. `100000 = 1.0` の固定小数点演算順序
2. 各乗算後の整数切り捨て
3. 敵師団リストの無作為な並べ替え
4. 攻撃師団ごとの攻撃対象リスト再構築
5. 攻撃配分ごとの別ダメージ処理
6. 攻撃回数の確率的丸め
7. 防御回数の確率的丸め
8. 対象師団に属する共有防御消費カウンター
9. 防御回数が残っている場合・切れた場合の命中判定
10. 命中ごとのSTR/ORG別ダイス判定
11. 装甲優位用の「対象Piercing対攻撃師団Armor」判定
12. 最終ダメージ用の「攻撃師団Piercing対対象Armor」段階判定
13. 現在戦力の10%単位切り捨て
14. 増援可能なreserve候補の抽出
15. reinforce chanceの二乗を基礎とする候補weight
16. weighted randomによる候補一個の選択
17. 選ばれた一個だけに対する最終増援抽選

期待値だけを比較する計算機なら上の近似式で十分だが、1時間単位の戦闘ログやセーブ状態と照合する決定論的シミュレーターでは、RNGを呼び出す順序まで合わせる必要がある。

## 18. 戦闘予備隊の増援確率

### 18.1 ここでいうreinforcement

この節で扱うreinforcementは、装備やmanpowerが師団へ補充されるreinforcementではない。陸戦へ登録されてreserveに待機している師団が、戦闘正面のactive combatへ入る判定を指す。

このMODの基礎値は次のとおりである。

```lua
REINFORCE_CHANCE = 0.02
SPEED_REINFORCEMENT_BONUS = 0.01
ARMY_INITIATIVE_REINFORCE_FACTOR = 0.25
```

`land_reinforce_rate`はtechnology、doctrine、trait、ideaなどから加算される。実際の候補師団ごとのchanceは、`FUN_140c2bf20`で概念的に次の要素から作られる。

```text
candidate_reinforce_chance
= base REINFORCE_CHANCE
+ land_reinforce_rate modifiers
+ Initiative由来の寄与
+ division speed由来の寄与
```

これに加えて、その師団が戦闘可能か、空いている戦闘幅または許容されるover-widthの範囲へ入れるかを確認する。条件を満たさずchanceが0になった師団は候補にならない。このため「reserve listに十個表示されている」ことと「抽選候補が十個ある」ことは必ずしも同じではない。

### 18.2 一時間の実際の処理

Ghidraで確認できる処理は次の形である。

```text
一時間ごとの各戦闘陣営について:
    active師団が使用しているcombat widthを集計
    空き幅がありreserveが存在する場合:
        eligible_candidates = []

        for division in reserves:
            p = calculate_reinforce_chance(division)
            if p > 0 and width条件を満たす:
                eligible_candidatesにdivisionを追加
                selection_weightをp²に近い値で設定

        selected = weighted_random(eligible_candidates)

        if random_percent < reinforce_chance(selected):
            selectedをreserve listから除去
            selectedをactive listへ追加
```

重要なのは、reserve listを走査するloopが存在していても、そのloop内で全師団へ成功判定を行っているわけではない点である。loopは候補とweightを作るためのものであり、最終的なreinforce chance抽選はweighted randomで選ばれた一師団だけに行われる。

`FUN_141261340`では候補ごとに`FUN_140c2bf20`を呼び、chanceからweightを作って配列へ保存する。その後、`FUN_14243de20`で一個のindexを選び、選ばれた師団について再び`FUN_140c2bf20`を呼んでchanceを取得し、次の比較を一回実行する。

```text
random_value % 100 < floor(reinforce_chance_fixed × 100 / 100000)
```

したがって、この最終判定はコード上では整数percentage point単位である。たとえば内部chanceが2.9%なら、当該比較に渡るthresholdは2となる。100%以上なら比較は常に成功する。

成功時はcallerの`FUN_141260340`が、返された一師団をreserve vectorから取り除き、active vectorへ一個だけ追加する。この関数は一時間ごとの陸戦処理`FUN_141265c00`から一回呼ばれる。そのため、この通常のreserve増援経路によって一陣営から同じ一時間に入る師団数は最大一個である。次の時間にも幅とreserveが残っていれば、同じ処理を改めて行う。

### 18.3 同じchanceのreserveが十個ある例

十個のreserveがすべて候補になり、全員のreinforce chanceが2%であると仮定する。weightも全員同じなので、最初の候補選択は各師団について10分の1となる。

```text
P(その時間に誰か一個が入る)
= 2%

P(特定の一個が入る)
= P(その一個が選ばれる) × P(選ばれた後に成功する)
= 1/10 × 2%
= 0.2%

P(同じ時間に二個以上が入る)
= 0%
```

十師団がそれぞれ独立に2%を振る方式なら、誰かが成功する確率は次のようになる。

```text
1 - (1 - 0.02)^10
≈ 18.3%
```

しかし実装は独立十回抽選ではないため、この18.3%にはならない。全候補のchanceが同じなら、reserveの数を増やしても最初の一師団が入る一時間当たり確率は2%のままである。

幅が空いたままで、chanceが変わらず、まだ一度も成功していないと仮定した場合、`h`時間以内に最初の一師団が入る確率は次の幾何分布で近似できる。

```text
P(first reinforcement within h hours)
= 1 - (1 - p)^h
```

2%なら24時間以内に一度以上成功する確率は約38.4%、最初の成功までの平均待ち時間は50時間である。ただし実戦では、戦闘終了、active師団の撤退、幅、tactic、modifierの変化などにより条件は一定ではない。

### 18.4 候補ごとのchanceが違う場合

候補ごとのchanceが異なる場合は、単純に全候補の平均chanceを使うわけではない。コードが候補選択に使用するweightは、固定小数点の切り捨てと最小値処理を除けば、おおむね次の形である。

```text
selection_weight_i ∝ p_i²
```

したがって、chanceの高い師団は二段階で有利になる。

1. weighted randomで最終判定の対象に選ばれやすい。
2. 選ばれた後の最終判定にも成功しやすい。

整数丸めを無視すれば、候補`i`がその時間に実際に増援する確率は概念的に次のように書ける。

```text
P(division i reinforces)
≈ (p_i² / Σp_j²) × p_i
= p_i³ / Σp_j²
```

その時間に誰か一個が入る総確率は、次のweighted averageになる。

```text
P(any reinforcement)
≈ Σp_i³ / Σp_i²
```

全員の`p_i`が等しければ、この式は元の`p`へ戻る。一方、高chance師団と低chance師団が混在すれば、高chance師団が結果を強く支配する。したがってreserve数そのものより、候補師団のreinforce chanceの分布が重要である。

### 18.5 実戦上の意味

- 同じchanceのreserveを十個置いても、一師団目の増援速度は十倍にならない。
- reserveが多いことには、次の時間以降も投入候補が残る、戦線維持の総量が増えるという価値はある。
- reinforce chanceを高める効果は、一時間ごとの成功率を直接上げる。
- chanceが高い師団は候補として選ばれやすいため、他のreserveより先にfrontへ入りやすい。
- active師団が全滅・撤退して戦闘が終了すれば、reserveが多数いても次の抽選機会を得られない。reinforce chanceは、いわゆるreinforce memeを防ぐうえで重要である。
- width条件により候補外になった師団は、その時間の抽選数を増やさない。

したがってreinforce chanceの最も正確な短い説明は、**「reserve全体について一時間に一個だけ選ばれる増援候補が、frontへ入れる確率。ただし候補選択自体も各師団のchanceを強く反映する」**となる。「各reserve師団が毎時間それぞれ振る確率」と理解すると、reserve数が多い戦闘で成功率を大きく過大評価する。
