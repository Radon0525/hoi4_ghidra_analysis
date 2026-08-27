# HOI4「innovative balance」陸軍偵察値・戦術選択の内部計算調査

調査対象：Workshop ID `820039020`、MODバージョン `13.00`、対応表記 `1.17.5.2`  
実行ファイル：ローカルの `hoi4.exe` を Ghidra で静的解析  
調査日：2026-08-25

## 結論

陸軍の `recon` は、Soft Attack・Defense・命中率・ダメージへ直接掛かる数値ではない。実コード上の主用途は、12時間ごとのCombat Tactic選択で使う「指揮官skill比較」に一律 `+5` を与えることである。

```text
各陣営の偵察代表値 R
= 戦闘に登録された師団の division recon の最大値

戦術選択score S
= 指揮官skill
 + 5  （自軍R > 敵軍Rのときだけ）
```

`R` が同値なら、どちらにも偵察bonusは付かない。偵察差が `0.1` でも `10` でも、勝った側が得るのは同じ `+5` である。

`S` が高い陣営は、相手の戦術が決まった後に自分の戦術を選ぶ。このとき相手戦術をcounterできる候補のweightだけが増える。

```text
ΔS = 高い側のS - 低い側のS

counter候補の最終weight
= 通常の最終weight × (1 + 0.35 × ΔS)
```

したがって、偵察優勢はcounterを保証しない。counter候補が使用可能である場合に、そのweighted randomの当選率を上げる効果である。

## 1. Ghidraで確認した実行経路

### 1.1 陣営の偵察値は参加師団の最大値

`FUN_141380ff0` (`0x141380ff0`) は、戦闘陣営オブジェクトの師団配列を走査する。

```c
side_recon = 0;
for (division in side.divisions) {
    division_recon = get_division_recon(division);
    if (side_recon < division_recon)
        side_recon = division_recon;
}
```

合計でも平均でもなく、**最大値**である。走査関数そのものには「戦闘正面にいる師団だけ」という除外条件がなく、戦闘陣営の配列に入っている師団をすべて比較する。このためreserve師団もreconを提供する実装になっている。

各師団の取得は `FUN_140c0ff80` (`0x140c0ff80`) で、師団の計算済み陸軍statsブロック `+0xd0` を返す。`STAT_ARMY_RECON` のmetadataと、この取得値を使うUI・戦術処理を照合してreconと特定した。

重要なのは二段階である。

```text
師団内：sub-unit・技術・教義・modifierからdivision reconを計算
戦闘内：両軍ともdivision reconが最大の一個師団だけを代表値にする
```

### 1.2 偵察優勢は指揮官skillへ一律加算

実戦の戦術更新関数は `FUN_141269900` (`0x141269900`) である。

1. 両軍の指揮官skillを `FUN_140d350f0` (`0x140d350f0`) で取得する。
2. `FUN_141380ff0` で両軍の最大reconを取得する。
3. 最大reconが高い側のskillへ `RECON_SKILL_IMPACT` を加算する。
4. 合成scoreが低い側から戦術を選ぶ。
5. 合成scoreが高い側は、相手の `countered_by` とscore差を受け取って後から選ぶ。

このMODでは：

```lua
RECON_SKILL_IMPACT = 5
```

よって式は次になる。

```text
S_A = leader_skill_A + (R_A > R_B ? 5 : 0)
S_B = leader_skill_B + (R_B > R_A ? 5 : 0)
```

Ghidraではdefineの実体が `DAT_14324ab40` で、`FUN_141269900` 内の二方向の加算命令まで確認した。UI用の比較関数 `FUN_14125b800` (`0x14125b800`) も同じ最大reconと同じdefineを使う。

### 1.3 高score側は「counter候補のweight」を増やす

戦術候補のweight構築は `FUN_14126a1e0` (`0x14126a1e0`)、実際のweighted randomによる選択は `FUN_141268ca0` (`0x141268ca0`) である。

`FUN_141269900` は低score側を通常抽選した後、その戦術の `countered_by` IDと `ΔS` を高score側の抽選へ渡す。候補IDが一致したときだけ：

```text
weight_multiplier = 1 + ΔS × INITIATIVE_PICK_COUNTER_ADVANTAGE_FACTOR
```

を適用する。このMODでは：

```lua
INITIATIVE_PICK_COUNTER_ADVANTAGE_FACTOR = 0.35
```

Ghidra上のdefine実体は `DAT_143246c38` である。コードは固定小数点 `100000 = 1.0` で、実際に `1.0 + ΔS × 0.35` を組み立ててcounter候補のweightへ掛けている。

最終的なcounter当選率は、counter候補の通常最終weightを `Wc`、その他の有効候補weight合計を `Wo` とすると：

```text
P(counter)
= Wc × (1 + 0.35 × ΔS)
  / [Wo + Wc × (1 + 0.35 × ΔS)]
```

ただし、次の場合は偵察優勢があってもcounterにならない。

- 相手戦術に `countered_by` が設定されていない。
- 指定されたcounter戦術が現在のphaseやtriggerを満たさない。
- weightは増えたが、weighted randomで別の戦術を引いた。

### 1.4 戦術更新間隔

`FUN_141265af0` (`0x141265af0`) は、戦術が未選択の場合、または戦闘hour counterが `TACTIC_SWAP_FREQUENCEY` の倍数の場合に `FUN_141269900` を呼ぶ。

このMODでは：

```lua
TACTIC_SWAP_FREQUENCEY = 12
```

したがって新規戦闘の最初と、その後12時間ごとの戦術再抽選時に、現在の最大reconと指揮官skillを比較する。

## 2. 指揮官skill差との関係

偵察優勢は「相手より必ず先手を取る」ではなく、指揮官skillへ `+5` する効果である。

| 自軍指揮官 | 敵指揮官 | 自軍だけ偵察優勢 | 合成score | 結果 |
|---:|---:|---:|---:|---|
| 4 | 4 | +5 | 9 対 4 | 自軍が後から選択、`ΔS=5` |
| 2 | 6 | +5 | 7 対 6 | 自軍が後から選択、`ΔS=1` |
| 1 | 6 | +5 | 6 対 6 | 同点、counter weight bonusなし |
| 1 | 7 | +5 | 6 対 7 | 敵軍が後から選択、`ΔS=1` |

つまり、自軍指揮官が相手より4level低くても偵察優勢で逆転でき、5level低いと同点、6level以上低いと逆転できない。

同格指揮官なら `ΔS=5` なので、counter候補のweightは：

```text
1 + 0.35 × 5 = 2.75倍
```

になる。ただしこれは当選確率が2.75倍になるという意味ではない。たとえば有効候補が四つ、通常weightがすべて同じで、そのうち一つだけがcounterなら：

```text
通常：       1 / 4 = 25.0%
偵察優勢後： 2.75 / (3 + 2.75) = 約47.8%
```

実戦では各戦術のbase、trigger、preferred tactic補正などが異なるので、この47.8%は仕組みを示す例にすぎない。

## 3. `initiative` という用語との区別

この処理は画面・localizationでleader initiativeのように表現されるが、師団statの `Initiative` とは別物である。

| 名称 | 実際の用途 |
|---|---|
| `recon` | 最大値比較に勝つとCombat Tactic用の指揮官skillへ `+5` |
| 指揮官skill / 戦術選択score | どちらが後から戦術を選ぶか、counter候補weightをどれだけ増やすか |
| 師団stat `Initiative` | Coordinationによる集中攻撃率やreinforce関連。今回のcounter式には入らない |
| `INITIATIVE_PICK_COUNTER_ADVANTAGE_FACTOR` | 名前にInitiativeがあるが、ここでは戦術選択score差1点あたりのcounter weight倍率 |

## 4. このMODのrecon供給源

### 4.1 標準recon系support company

`common/units/recon.txt` の基礎値は次のとおり。

| sub-unit | 基礎recon | `category_recon` | 1939技術後 | 1942技術後 | 1944技術後 |
|---|---:|:---:|---:|---:|---:|
| `recon` | 1.0 | あり | 3.0 | 5.0 | 7.0 |
| `mot_recon` | 1.5 | あり | 3.5 | 5.5 | 7.5 |
| `armored_car_recon` | 2.0 | あり | 4.0 | 6.0 | 8.0 |
| `light_tank_recon` | 1.0 | あり | 3.0 | 5.0 | 7.0 |

`tech_recon2`、`tech_recon3`、`tech_recon4` は、それぞれ：

```lua
category_recon = { recon = 2 }
```

を与えるため、研究済み効果は累積する。基礎値だけでなく、研究時点でも `armored_car_recon` が標準四種の中で最大である。

### 4.2 reconを持つが標準技術bonusを受けないsub-unit

次のsub-unitにも生のreconがあるが、定義上 `category_recon` を持たない。そのため上記三技術の `category_recon = { recon = 2 }` は付かない。

| sub-unit | 基礎recon | 備考 |
|---|---:|---|
| `airborne_light_armor` | 1.0 | `category_recon` なし |
| `long_range_patrol_support` | 2.0 | `category_recon` なし |
| `helicopter_recon` | 2.0 | `category_recon` なし |
| `motorized_military_police` | 1.0 | `category_recon` なし |
| `helicopter_brigade` | 1.0 | `category_recon` なし |

これはこのMODのデータ上かなり重要である。たとえば1944年の `helicopter_recon = 2` は名前上は高性能reconに見えるが、標準recon技術を三つ取った `armored_car_recon = 8` よりかなり低い。別の技術・教義・複数sub-unitの加算がない同条件なら、戦術用reconだけでは標準四種に負ける。

### 4.3 教義・modifier

このMODには、選択した陸軍subdoctrineに応じて `category_recon` へ追加reconを与えるものがある。例として `operations_subdoctrines.txt` には `recon = 1`、別経路には `recon = 0.1` がある。これらは全国家共通の固定値ではなく、選択した経路で変わる。

また `recon_factor` は最終division reconを割合補正するmodifierで、このMODでは例として次がある。

- `trickster`: `recon_factor = 0.25`
- `promoted_from_the_ranks`: `recon_factor = 0.05`
- `flexible_organization_spirit`: `recon_factor = 0.25`

戦闘コードは、これらを含むstats計算を終えたdivision reconを読み、その最終値の最大を比較する。

## 5. 実用上、偵察値はどの程度重要か

結論は「強いことはあるが、限界効用が極端にbinary」である。

### 価値が高い状況

- reconを少し増やすことで敵軍の最大reconを上回れる。
- 指揮官skill差が5以内で、`+5` によって戦術選択順を同点または逆転できる。
- 相手が強いがcounter可能な戦術を多く使う。
- 自軍のcounter戦術が現在のphase・triggerで使用可能になっている。
- 長期戦で12時間ごとの再抽選を何度も行う。

### 価値が低い状況

- 増やしても敵の最大reconを超えない。
- すでに十分上回っており、追加しても勝敗が変わらない。
- 指揮官skill差が大きすぎて `+5` でも合成scoreを逆転できない。
- 相手戦術に有効なcounterがない、または自軍がcounter戦術のtriggerを満たさない。
- 戦闘が12時間未満で終わり、再抽選回数が少ない。

### 編成上の意味

戦闘陣営では最大値しか見ないので、全師団へ均等にreconを付ける必要はない。理論上は、一個の高recon師団を戦闘へ登録しておけば、その値が陣営全体を代表する。reserveもreconを提供するため、高reconの予備師団を置く運用も成立する。

ただし、その一個師団が戦闘から離脱・撤退すれば次点へ落ちる。実戦では複数師団に持たせることに、直接的な加算効果ではなく「代表値を失わない冗長性」がある。

## 6. 主要関数・define対応

| 内容 | Ghidra上の関数・global |
|---|---|
| division recon取得 | `FUN_140c0ff80` (`0x140c0ff80`) |
| 陣営内の最大recon集約 | `FUN_141380ff0` (`0x141380ff0`) |
| 指揮官skill取得 | `FUN_140d350f0` (`0x140d350f0`) |
| 最大recon比較と戦術選択順 | `FUN_141269900` (`0x141269900`) |
| 戦術候補weight構築 | `FUN_14126a1e0` (`0x14126a1e0`) |
| weighted randomによる戦術確定 | `FUN_141268ca0` (`0x141268ca0`) |
| 戦術更新の12時間判定 | `FUN_141265af0` (`0x141265af0`) |
| UI用の両軍score比較 | `FUN_14125b800` (`0x14125b800`) |
| `RECON_SKILL_IMPACT` | `DAT_14324ab40` |
| `INITIATIVE_PICK_COUNTER_ADVANTAGE_FACTOR` | `DAT_143246c38` |
| `TACTIC_SWAP_FREQUENCEY` | `DAT_14324682c` |

## 7. 確度

### Ghidra実行コードとMODデータから確認済み

- division reconの取得経路
- 陣営内で平均・合計ではなく最大値を使うこと
- reserveを除外する条件が最大値集約関数にないこと
- 高recon側にだけ `RECON_SKILL_IMPACT = 5` を一律加算すること
- 偵察差の大きさ自体を倍率へ使わないこと
- 合成scoreの低い側を先、高い側を後に抽選すること
- 高score側のcounter候補weightが `1 + 0.35 × ΔS` 倍になること
- counterは強制選択ではなくweighted randomであること
- 戦術再抽選が12時間ごとであること
- MOD内の各sub-unit基礎recon、標準recon技術の累積 `+2 / +2 / +2`
- 一部の名前上recon系sub-unitに `category_recon` がなく、標準技術bonusを受けないこと

### 注意点

Ghidra解析はこのローカル `hoi4.exe` とMODデータの組み合わせに対する静的解析である。ゲーム本体の別buildやMOD更新でdefine、関数配置、sub-unit categoryが変われば結果も変わり得る。
