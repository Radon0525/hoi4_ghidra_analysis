# HOI4 海戦内航空戦の内部処理

> **innovative balance v13.00 対応版**  
> 係数は Workshop ID `820039020` の `common/defines/00_defines.lua` を基準にしている。実行ファイル側の処理構造・関数アドレスは元のGhidra解析結果、数値はMOD値である。

## 0. 調査範囲と結論

提供された Ghidra プロジェクト内の `hoi4.exe` を対象に、文字列として残っている

`source\naval\naval_fire_exchange_air.cpp`

の関数群と、その呼出元・define 登録関数を逆コンパイルして追跡した。

結論から言うと、海戦をめぐる航空処理は単一の空戦関数ではなく、少なくとも次の3層に分かれている。

1. 戦略地域側の通常空戦。基地戦闘機はここで海軍攻撃・港湾攻撃中の爆撃機を撃墜・妨害できる。
2. 海戦中の空母航空隊同士の空対空戦闘。通常空戦コアを再利用するが、双方が空母戦闘用航空隊なら撃墜scaleが3倍になる。
3. 艦艇を攻撃する直前の海戦専用strike。防御側艦載CAP、目標艦AA、爆撃を一方向に解決する。

このうち `source\naval\naval_fire_exchange_air.cpp` が担当するのは主に3番目である。海戦側に専用の「航空攻撃交換」オブジェクトを作り、概ね次の順番で一方向のstrikeとして解決する。

1. 艦載機または基地航空隊を海戦へ登録
2. 過密・艦載機発艦効率から今回使える攻撃側を作る
3. 重み付き乱数で攻撃目標を1隻選ぶ
4. 防御側空母の戦闘機が攻撃機を迎撃
5. 選ばれた艦自身の対空砲が攻撃機を撃墜
6. 生き残った攻撃機のうち、乱数で「攻撃成功機数」を決める
7. 艦自身＋艦隊全体AAによるダメージ軽減を適用
8. 艦の Strength と Organization にダメージを与え、クリティカル・撃沈・戦果を記録

したがって「通常の地域空戦でまず航空任務同士が交戦し、それを通過して海戦へ参加した攻撃機には、別の小さな航空攻撃解決器がもう一度走る」という理解が最も近い。以下でいう「一方向」とは、この艦艇攻撃直前のstrike処理を指し、海戦をめぐる航空戦全体に戦闘機同士の相互戦闘が存在しない、という意味ではない。

「空母艦載機が海戦中に機能しない」という報告についても再検証した。攻撃機には夜間・吹雪・低Org・任務・目標・クールダウン・整数丸めによる正常な停止条件が複数ある。一方、innovative balance v13.00には、5段階の空母防衛スタンスをすべて攻撃100%にする定義があり、艦艇攻撃直前の艦載戦闘機CAPを実質0にする。後者はexeの偶発的不具合ではなく、MOD値と未変更localizationの不整合である。詳細は第18節にまとめる。

以下では `100000 = 1.0` のゲーム内部固定小数を、通常の実数に直して表記する。特に断らない限り `clamp(x,0,1)` は 0～1 への制限、`round` は正数に対する四捨五入である。

---

## 1. 処理全体の入口

中心関数は次の通り。

| 役割 | アドレス |
|---|---:|
| 海戦側の毎時航空更新 | `FUN_141bc2d00 @ 0x141bc2d00` |
| 1回の航空攻撃全体 | `FUN_1418db000 @ 0x1418db000` |
| 攻撃機の性能集計 | `FUN_1418d83e0 @ 0x1418d83e0` |
| 空母発艦効率の本体 | `FUN_1418d6310 @ 0x1418d6310` |
| 空母Orgを含む最終発艦効率 | `FUN_1418d7c40 @ 0x1418d7c40` |
| 攻撃目標の重み付き抽選 | `FUN_1418d9e40 @ 0x1418d9e40` |
| 艦載戦闘機による迎撃 | `FUN_1418daad0 @ 0x1418daad0` |
| 目標艦自身のAAによる撃墜 | `FUN_1418dc060 @ 0x1418dc060` |
| 艦隊AAを含む爆撃ダメージ軽減 | `FUN_1418cebf0 @ 0x1418cebf0` |
| 爆撃命中数・STR/Orgダメージ | `FUN_1418da3c0 @ 0x1418da3c0` |
| 撃墜機を実際の航空隊から除去 | `FUN_1418d98f0 @ 0x1418d98f0` |
| 空母攻撃後の再攻撃待ち時間 | `FUN_1418dcd60 @ 0x1418dcd60` |
| 戦略地域側の航空戦参加者収集 | `FUN_140beee30 @ 0x140beee30` |
| 通常空戦の対象別割当 | `FUN_140f40320 @ 0x140f40320` |
| 通常空戦・空母戦の撃墜計算 | `FUN_140f4a790 @ 0x140f4a790` |

`FUN_141bc2d00` は海戦の航空攻撃オブジェクトを走査し、機数と性能を再集計してから `FUN_1418db000` を呼ぶ。攻撃が成立した空母航空隊には、その後 `FUN_1418dcd60` で次回攻撃までの待ち時間が設定される。

一方、`FUN_140beee30` は `source\strategicair.cpp` 側にあり、地域内の通常航空隊と海戦中の空母航空隊を空対空戦闘用リストへ整理する。実際の相互撃墜計算は `source\airmission.cpp` の `FUN_140f4a790` が担当する。この経路は `naval_fire_exchange_air.cpp` のstrike処理とは別である。

---

## 2. 艦載機と基地航空隊は別経路

### 2.1 艦載機

空母を含む海戦側が作られると、`FUN_141bc0ee0` → `FUN_1418d90f0` により空母上の航空隊が専用オブジェクトへ登録される。

コンストラクタ `FUN_1418d5b00` は、空母・海戦側オブジェクトへの参照がある場合を「内部＝艦載機」として扱う。これは海戦中に持続し、攻撃後も再使用される。

艦載機のMOD攻撃間隔は次の define。

```lua
NAir.CARRIER_HOURS_DELAY_AFTER_EACH_COMBAT = 4
```

実際には `FUN_1418dcd60` が、この4時間に空母側の sortie-delay 系 modifier を加え、端数を切り上げて次回待ち時間へ格納する。

### 2.2 戦略地域側から登録される外部航空隊（external planes）

戦略地域側の任務処理から来る航空隊は `FUN_1415a9410` → `FUN_1418d8d50` で「外部航空隊」フラグ付きの一回用オブジェクトとして登録される。典型例は陸上基地のNaval Strikeだが、分類を決めるのは機体種や設計ではなく**戦略地域側から外部参加として渡されたこと**である。そのため、空母を発進基地とする航空隊が戦略地域任務として渡される場合も、この外部経路を通りうる。

このオブジェクトは1回処理されると processed フラグが立ち、内部艦載機用の4時間クールダウンでは再利用されない。再度来るかどうかは通常の航空任務側のスケジューラに依存する。通常航空戦側のMOD待ち時間defineは次の通り。

```lua
NAir.HOURS_DELAY_AFTER_EACH_COMBAT = 6
```

戦略地域側では通常この値を往復分として2倍して使うため、基地航空隊の地域任務処理機会はMODでは12時間周期になる。

外部航空隊には、海戦へ入れる機数の上限がある。`FUN_1415a44a0` へMOD値を入れた式は概ね次の通り。

```text
battleDays = floor(海戦継続時間 / 24時間)

externalCap = max(
    20,
    trunc(totalShipCurrentStrength × 0.025 × (1 + battleDays × 0.0))
)

= max(20, trunc(totalShipCurrentStrength × 0.025))
```

対応する define は次の3個。

```lua
NAVAL_COMBAT_EXTERNAL_PLANES_JOIN_RATIO         = 0.025
NAVAL_COMBAT_EXTERNAL_PLANES_JOIN_RATIO_PER_DAY = 0.0
NAVAL_COMBAT_EXTERNAL_PLANES_MIN_CAP            = 20
```

`FUN_1415a9410` は、すでに海戦へ割り当てた外部機数をこの上限から引き、今回要求された機数との小さい方だけを新たに参加させる。MODでは日数係数が0なので、海戦が長期化しても外部航空機上限は増えない。艦隊の現在総Strengthの2.5%と最低20機だけで決まる。

### 2.3 基地戦闘機は海戦内CAPへ直接参加しない

外部航空隊の初期化 `FUN_1418d8d50` は航空隊を全航空隊リストへ登録した後、次の任務マスクを満たす航空隊だけを艦艇攻撃リストへ追加する。

```text
if ((airWingMissionMask & 0x190) != 0)
    addToNavalStrikeList()
```

`0x190` は海軍攻撃・港湾攻撃・神風系の任務群であり、通常の制空・迎撃任務は含まれない。したがって、基地海軍爆撃機は外部攻撃隊として海戦へ入れるが、通常の基地戦闘機は海戦専用strikeの攻撃側にも防御側CAPにも入らない。

防御側CAPは `FUN_1418db000` が防御側艦隊の空母を走査し、`FUN_140f295f0` の戦闘機能力判定を満たす搭載航空隊だけを集めて作る。基地戦闘機をこのCAPリストへ加える経路は確認できない。

ただし基地戦闘機が海軍攻撃を妨害できないわけではない。基地戦闘機は海戦の外側にある通常の戦略地域空戦で、海軍攻撃・港湾攻撃中の爆撃機を撃墜または任務妨害できる。その結果、海戦や港湾攻撃へ到達できる爆撃機が減る。Wiki等でいう「敵戦闘機が目標艦船へ到達する前に爆撃機を妨害する」はこの上流段階を指しており、海戦内CAPへの参加を意味しない。

```text
基地戦闘機による通常空戦
    ↓ 撃墜・任務妨害
生き残って攻撃可能な基地爆撃機
    ↓ 外部航空隊として登録
艦載CAP → 目標艦AA → 爆撃
```

---

## 3. 海戦内の航空過密ペナルティ

`FUN_1415a7860` が海戦内で同時に扱う航空機の過密係数を計算する。

まず目標側の艦種から「航空機を受け入れられる戦場の大きさ」に相当する silhouette 上限を求める。艦種別係数はMODでも次の値である。

```lua
SHIP_SILHOUETTE_VALUE_PLANES_CAPITAL    = 10
SHIP_SILHOUETTE_VALUE_PLANES_SCREEN     = 5
SHIP_SILHOUETTE_VALUE_PLANES_CARRIER    = 16
SHIP_SILHOUETTE_VALUE_PLANES_SUPPORT    = 3
SHIP_SILHOUETTE_VALUE_PLANES_CONVOY     = 4
SHIP_SILHOUETTE_VALUE_PLANES_SUBMARINE  = 7
SCREEN_CAP_REDUCTION_FACTOR             = 0.02
```

軽快艦の silhouette は主力艦数に応じた `1 / (1 + capitals × 0.02)` 型の補正を受ける。この値とMODの固定下限10000の大きい方が、過密ペナルティなしで扱える機数 `L` になる。

```text
L = max(10000, targetFleetSilhouettePlaneCapacity)
```

海戦内の攻撃参加機数を `N` とすると、`N > L` の場合の係数は次の通り。

```text
stackingEfficiency = 1 / (1 + (N - L) × 0.005)
```

`N <= L` なら 1.0。

対応 define:

```lua
NAVAL_COMBAT_PLANE_MIN_STACKING_PENALTY = 10000
NAVAL_COMBAT_PLANE_STACKING_PENALTY_EFFECT = 0.005
```

この係数は `FUN_1418db000` に渡され、今回性能集計へ入る攻撃航空隊群を減らす。ただしMODでは無補正枠が10000機なので、通常規模の海戦では事実上stackingペナルティが発生しない。10000機を超えた場合だけ、分母が大きくなる逓減型の式が働く。

---

## 4. 艦載機の発艦数と空母Organization

艦載機だけは `FUN_1418d7c40` で発艦効率を計算する。基地航空隊の一回用オブジェクトはこの空母発艦効率分岐を通らない。

大枠は次の積である。

```text
finalSortieEfficiency
  = offensiveStanceRatio または baseCarrierEfficiency
  × crowding/capacity factor
  × weather factor
  × night factor
  × carrier/country/commander modifiers
  × carrierOrgRatio
```

海戦開始時の選択スタンスと基礎値は次のdefineで決まる。

```lua
CARRIER_OFFENSIVE_STANCE_SORTIE_RATIO = { 1.0, 1.0, 1.0, 1.0, 1.0 }
CARRIER_OFFENSIVE_STANCE_DEFAULT_INDEX = 2
SELECTED_SORTIE_INITIAL_TIME = 24
BASE_CARRIER_SORTIE_EFFICIENCY = 0.8
```

`FUN_1418d6310` は、海戦開始後24時間は選択されたスタンスの攻撃出撃比率を読み、タイマーが0になった後は `BASE_CARRIER_SORTIE_EFFICIENCY` を使う。バニラはスタンステーブルが `{0, 0.25, 0.50, 0.75, 1.0}`、基礎値が0.50であり、既定の「バランス」0.50と24時間後の0.50が一致する。MODは全スタンス1.00、基礎値0.80なので、晴天・昼間・Org100%等を固定しても、攻撃出撃の基礎部分が海戦開始後24時間だけ1.00、その後0.80へ低下する。

防御出撃比率はdefineコメントと実装の両方で次になる。

```text
defensiveStanceRatio = 1 - offensiveStanceRatio
```

したがってMODの全スタンス1.00は、完全防御・慎重・バランス・活動中・攻撃的のどれを選んでも、防御出撃比率を0にする。これは第7節の直接CAPへ実際に渡される。

天候と夜間も単なる表示補正ではなく、この積を0まで下げられる。MODのstatic modifierはバニラと同じである。

| 条件 | `carrier_traffic` | 無軽減時の当該係数 |
|---|---:|---:|
| 豪雨 | -0.80 | 0.20 |
| 吹雪 | -1.00 | 0.00 |
| 完全な夜間 | -1.00 | 0.00 |

天候係数と夜間係数は別々に `0～1` へ制限される。`carrier_traffic` や `carrier_night_traffic` の正のbonusがあれば部分的に相殺できるが、無補正の完全な夜間または吹雪では攻撃出撃数は0になる。豪雨では20%まで落ち、さらにOrg・甲板過密・その他補正を掛けてから機数へ丸めるため、小航空隊では0機になりやすい。

最後の `carrierOrgRatio` は、前回調査で特定した `currentOrg / maxOrg` そのもの。`FUN_140bcb030` の戻り値を発艦効率へ直接乗算している。

今回発艦する機数は概ね次の形。

```text
sortiePlanes = round(attachedStrikePlanes × clamp(finalSortieEfficiency, 0, 1))
```

したがって空母Orgは「爆撃1機当たりの命中率」を直接下げるのではなく、まず出撃できる機数を線形に減らす。たとえば他の補正が同一なら、Org 50%の空母はOrg 100%の空母に対して発艦効率が半分になる。積の後で整数機数へ丸めるため、低Orgと豪雨等が重なると連続的な効率低下が最終段で0機という不連続な結果になる。

また、`FUN_1418db000` は攻撃航空隊の発進基地を `CShip` として解釈できる場合、その艦を現在の海戦参加グループから `FUN_141bc04e0` で検索する。発進元空母が**この海戦の参加艦として実際に見つかった場合だけ**、後述するNaval Attackに次が掛かる。

```lua
NAir.NAVAL_STRIKE_CARRIER_MULTIPLIER = 1.00
```

現在のバニラ値は10.0、innovative balance v13.00は1.00である。このMODでは条件を満たしてもNaval Attackの追加ダメージ倍率は実質無効化されている。空母の発艦効率・Org・攻撃間隔など、他の空母専用処理は引き続き存在する。

---

## 5. 攻撃機の性能集計

`FUN_1418d83e0` は参加航空隊から、少なくとも次の3値を機数加重で集計する。

```text
A = average internal naval_strike_attack
T = average internal naval_strike_targetting
G = average naval_strike_agility
```

ここで重要なのは、設計画面の表示値をそのまま使わないことである。Naval Attack getter `FUN_140f26580` とNaval Targeting getter `FUN_140f26910` は、任務別のstat補正を適用した後、表示スケールを0.1倍して返す。

```text
A_internal = displayedNavalAttack    × 0.1 × attackStatModifiers
T_internal = displayedNavalTargeting × 0.1 × targetingStatModifiers
```

たとえば設計画面の「対艦攻撃26」「対艦照準16.5」は、追加補正を無視すれば、それぞれ内部値2.6と1.65としてstrike式へ入る。

任務固有statも無視されない。`FUN_1418d83e0` は通常Naval Strike／Kamikazeではマスク `0x90`、Port Strikeでは `0x100` を使って参加資格を確認し、航空隊の `wing+0xa4` に保存された実際の任務ビットを上記getterへ渡す。したがって、Naval Strikeイベントとして収集された航空隊が、getterだけ「任務なし」として対艦攻撃1を読む、という通常経路にはなっていない。

装備性能だけでなく、航空隊・国家・任務・空母関連modifierがこの段階で反映される。外部航空隊では、海戦への登録時に保存された任務効率系係数が攻撃値へ追加で掛かる。

以下の式に出てくる `A`, `T`, `G` は、単純な設計画面の素の値ではなく、この集計後の内部値である。

---

## 6. 攻撃目標の選択

`FUN_1418d9e40` は、攻撃可能な各艦にスコアを付け、最後に重み付き乱数で1隻を選ぶ。

対象艦 `i` について、次を定義する。

```text
M_i  = 最大Strength
S_i  = 現在Strength / 最大Strength
AA_i = 対空値
E    = 防御側screening efficiency（0～1）
```

艦種係数は次の線形補間。

```text
classFactor_i
  = E × TARGET_BASE[class]
  + (1 - E) × TARGET_SCALE[class]
```

MOD値:

| 艦種 | BASE（screening 100%側） | SCALE（screening 0%側） |
|---|---:|---:|
| 潜水艦 | 10 | 10 |
| 軽快艦 | 10 | 10 |
| 主力艦 | 10 | 50 |
| 空母 | 10 | 10 |
| 輸送船 | 1 | 10 |

最終的な重みは、通常艦では次の形になる。

```text
weight_i =
    M_i
  × classFactor_i
  × (1 + (1 - S_i) × 5)
  + clamp(5 - AA_i, 0, 5) × 12.5
```

対応 define:

```lua
NAVAL_COMBAT_AIR_STRENGTH_TARGET_SCORE = 5.0
NAVAL_COMBAT_AIR_LOW_AA_TARGET_SCORE   = 12.5
```

重要な性質は次の通り。

- screening が崩れるほど主力艦の重みは増えるが、空母はBASEとSCALEがともに10なのでscreeningによる特別な増加を受けない。輸送船は1から10へ増える。
- 最大Strengthが大きい艦ほど元の重みが大きい。
- 損傷している艦ほど最大6倍まで重みが上がる。
- AAが5未満なら、低AAボーナスが `12.5` 倍で別加算される。AA 5以上ではこの加算は0。
- 最も重い艦を確定で選ぶのではなく、重みに比例した抽選である。

乱数は `naval_fire_exchange_air.cpp:0x43d` で1回振られ、`FUN_14243de20` が重み付きインデックスを返す。

---

## 7. 艦艇攻撃直前の防御第1段階：艦載戦闘機の迎撃

目標側の空母に属する fighter 航空隊を集め、`FUN_1418daad0` で攻撃機を一方向に迎撃する。

この直接CAPの収集は、現在選択中の迎撃・制空任務ビットではなく、`FUN_140f295f0` による装備のfighter能力判定を中心に行う。したがって、艦艇到達直前のCAPだけを見る限り、「艦上戦闘機に迎撃任務を出していないから必ず不参加」とは読めない。一方、次節の通常空母航空隊同士の空戦へ入るかどうかは戦略地域・任務側の参加条件を通るため、両処理を同じ任務判定として扱ってはいけない。

次を定義する。

```text
F = 防御側艦載戦闘機数
R_off = 選択スタンスの攻撃出撃比率
R_def = 1 - R_off
B = 来襲攻撃機数
AF = 防御戦闘機の平均 Air Attack
DF = 来襲機の平均 Air Defence
```

`FUN_1418db000` は選択スタンスの値を直接読み、防御側について `1 - R_off` を作る。`FUN_1418daad0` は各戦闘機航空隊の機数へこの比率を掛けて合計する。その後、海戦専用の35%係数が掛かる。

```text
F_stance = F × R_def
F_defend = F_stance × 0.35
```

さらに1機の来襲機に集中できる機数は最大1機相当。

```text
F_engaged = min(F_defend, B × 1.0)
```

基礎ダメージ:

```text
D0 = F_engaged × 0.30 × AF
```

対応 define:

```lua
CARRIER_PERCENTAGE_DEFEND                 = 0.35
COMBAT_MULTIPLANE_CAP                     = 1.0
CARRIER_COMBAT_DAMAGE_STATS_MULTIPLIER    = 0.30
```

innovative balance v13.00は `CARRIER_OFFENSIVE_STANCE_SORTIE_RATIO` の5要素がすべて1.00なので、全スタンスで `R_def = 0` となる。従って `CARRIER_PERCENTAGE_DEFEND = 0.35` は0へ掛かるだけで、防御CAPを復活させない。戦闘機航空隊リスト自体が存在する場合、後段の期待撃墜数には最低値0.001があるため、ごく低確率の端数切上げで1機撃墜が出る余地はあるが、通常の意味での戦闘機数・Air Attackに比例したCAPは失われる。

MODはこのスタンステーブルに対応する日本語localizationを上書きしていない。ゲーム表示は「完全防御＝すべての航空機を防御行動に使用」「バランス＝50%を防御」と説明する一方、実値はどちらも攻撃100%・防御0%である。これはUIだけの誤表示ではなく、実計算に0が渡る設定不整合である。

この `D0` に、通常空戦でも使われる次の共通ヘルパーが適用される。

- `FUN_140f3bb90`: agility差によるダメージ減少
- `FUN_140f3f240`: speed差によるダメージ増加

概略式は次の通り。

```text
D = D0 + speedBonus(D0) - agilityPenalty(D0)

expectedFighterKills = max(0.001, 0.01 × D / DF)
```

端数は `naval_fire_exchange_air.cpp:0x27b` の `random_fixed` で確率的に切り上げられる。

```text
kills = floor(expected)
      + Bernoulli(frac(expected))
```

実際の除去数は `FUN_1418d98f0` 側で残存攻撃機数までに制限される。

この関数内では防御戦闘機側の損失は計算されない。通常空戦のように両側が相手を撃つdogfightではなく、「今回のstrikeへ防御側戦闘機が損害を与える」一方向処理である。ただし、これは `FUN_1418daad0` 単体についての結論であり、次節の空母航空隊同士の空対空戦闘は別に存在する。

`COMBAT_DAMAGE_SCALE_CARRIER = 3.00` はこの `FUN_1418daad0` からは参照されない。艦艇攻撃直前のCAPには上記の海戦専用式が使われ、`COMBAT_DAMAGE_SCALE_CARRIER` は次節の空母航空隊同士の空対空戦闘で使われる。

---

## 8. 空母航空隊同士の空対空戦闘

海戦中の空母航空隊同士の空対空戦闘は、海戦専用strike関数ではなく、`source\strategicair.cpp` と `source\airmission.cpp` の通常空戦系を通る。

`FUN_140beee30` は航空戦参加エントリを作る際、航空隊の基地が `CShip` で、その艦船が海戦中であることを示す周辺状態を満たす場合に空母戦闘用フラグを付ける。`FUN_140f40320` は攻撃側と対象側の両エントリを比較し、概ね次の条件を `FUN_140f4a790` へ渡す。

```text
carrierCombatPair =
    attackerEntry.carrierCombat
    && targetEntry.carrierCombat
```

`COMBAT_DAMAGE_SCALE_CARRIER` のdefine登録以外の実使用参照は `FUN_140f4a790` の1か所である。

```text
0x140f4aa52  COMBAT_DAMAGE_SCALE を読み込む
0x140f4aa5c  carrierCombatPair != 0 なら
             COMBAT_DAMAGE_SCALE_CARRIER に差し替える
```

通常空戦コアの式を次の記号で表す。

```text
C = 交戦機数、平均Air Attack、対象配分・距離等から作る攻撃量
B = C × COMBAT_DAMAGE_STATS_MULTILPIER
D = 対象側の平均Air Defence
```

速度差と機動性差を反映した有効ダメージは、概ね次の形になる。

```text
effectiveDamage = B + speedBonus(B) - agilityPenalty(B)
```

最終的な期待撃墜数は次の通り。

```text
expectedKills = max(
    0.001,
    0.01 × damageScale × effectiveDamage / D
)

damageScale = COMBAT_DAMAGE_SCALE_CARRIER  if carrierCombatPair
            = COMBAT_DAMAGE_SCALE          otherwise
```

MOD値は次の通り。

```lua
COMBAT_DAMAGE_STATS_MULTILPIER = 0.2
COMBAT_DAMAGE_SCALE            = 1.00
COMBAT_DAMAGE_SCALE_CARRIER    = 3.00
```

比較すると、現在のバニラは `COMBAT_DAMAGE_SCALE_CARRIER = 5.00` である。したがってこのMODは、通常空戦scaleを1.00のまま維持しつつ、空母航空隊同士の追加致死率を5倍から3倍へ弱めている。

したがって、他の条件が同じなら、双方が空母戦闘用航空隊である空対空戦闘の期待撃墜数は通常空戦の3倍になる。これは「空母機の被撃墜率」だけへ一方的に掛かる脆弱性係数でも、迎撃開始確率やCAP投入率でもない。現在攻撃側として計算されている航空隊の撃墜力を3倍にするため、両軍がそれぞれ射撃側になる空戦全体では双方の損害を大きくする対称的な致死率倍率である。

期待値の小数部分は通常空戦と同様に同期乱数で確率的に丸められ、最終撃墜数は交戦可能な対象機数で制限される。

片側だけが通常の基地航空隊である組合せでは `carrierCombatPair` が成立せず、空戦コアへその組合せが到達した場合のscaleは通常値1になる。ただし基地戦闘機は海戦専用CAPへ入るわけではなく、これは戦略地域側の通常空戦として処理される。

---

## 9. 防御第2段階：目標艦自身のAAによる撃墜

戦闘機迎撃後の残存機に対して `FUN_1418dc060` が走る。

ここで使うAAは、原則として選ばれた目標艦自身のAAである。後述する「艦隊全体AAによる爆撃ダメージ軽減」とは別処理。

```text
Q  = 目標艦の anti-air targeting 基礎値 = 0.7
G  = 攻撃機の平均 naval_strike_agility
AA = 目標艦の対空攻撃値
N  = 戦闘機迎撃後の攻撃機数
```

まず targeting を航空機の agility で補正する。

```text
qAdjusted = max(0.01, Q + 0.5 × (Q - G))
aaEngageChance = qAdjusted × 0.65
```

対応 define:

```lua
ANTI_AIR_TARGETING               = 0.7
ANTI_AIR_TARGETTING_TO_CHANCE    = 0.65
```

`naval_fire_exchange_air.cpp:0x210` で `U <= aaEngageChance` の判定を行う。失敗すれば、この攻撃回の艦対空撃墜は0。

成功した場合の撃墜割合:

```text
aaKillFraction = clamp(AA × 0.0006, 0, 1)
expectedAAKills = N × aaKillFraction
```

```lua
ANTI_AIR_ATTACK_TO_AMOUNT = 0.0006
```

端数は `naval_fire_exchange_air.cpp:0x21a` の別の一様乱数で確率的に丸める。

このため艦AAには二段階のランダム性がある。

1. 今回AAが有効に命中するか
2. 期待撃墜数の小数部分を切り上げるか

### 神風攻撃の特例

来襲航空隊に kamikaze フラグがある場合、この関数は明示的に次を行う。

- AA値へ `BASE_KAMIKAZE_DAMAGE = 2.0` を加算
- AA targetingへ `BASE_KAMIKAZE_TARGETING = 2.0` を加算
- 最初のAA命中判定をスキップし、必ずAA撃墜計算へ進む
- 撃墜数を最低1機にする
- 最後に `AIR_NAVAL_KAMIKAZE_LOSSES_MULT = 4.0` 倍する

したがって神風隊は、この防御段階では通常の攻撃隊より非常に大きな自己損耗を受ける。

---

## 10. 爆撃成功機数

戦闘機と艦AAを通過した残存機を `N` とする。

```text
T    = 平均 naval_strike_targetting
M_T  = 海戦側・指揮官・国家などのtargeting modifier
```

成功係数は次の通り。

```text
pStrike = clamp(T × M_T × 0.30, 0, 1)
```

```lua
NAVAL_STRIKE_TARGETTING_TO_AMOUNT = 0.3
```

ここで重要なのは、各機について独立に命中判定をしていないこと。`naval_fire_exchange_air.cpp:0x45b` で攻撃隊全体に対して一様乱数 `U ∈ [0,1)` を1個だけ振る。

```text
successfulPlanes = round(N × pStrike × U)
```

`round` の実体は `FUN_14243b2f0 @ 0x14243b2f0` で、正数なら `floor(x + 0.5)`。

したがって同じ `N` と `pStrike` でも、成功機数は0付近から `N × pStrike` 付近まで一様に大きく振れる。二項分布ではない。平均的には概ね `N × pStrike / 2` になる。

成功機数が四捨五入後に0なら、その回は艦へのSTR/Orgダメージなしで終了する。

---

## 11. 艦隊AAによる爆撃ダメージ軽減

ここは「航空機を撃墜するAA」と別式である。

海戦側更新 `FUN_1415a72c0` が、参加艦すべてのAAを合計し、艦隊AA成分をあらかじめ保存する。

```text
fleetAAComponent
  = fleetAAEfficiency
  × sum(all participating ships' AA)
  × 0.40
```

```lua
SHIP_TO_FLEET_ANTI_AIR_RATIO = 0.40
```

選ばれた目標艦のAAをもう一度加えて、爆撃ダメージ軽減に使う実効AAを作る。

```text
effectiveAA = targetShipAA + fleetAAComponent
```

`FUN_1418cebf0` の軽減率は define コメント通り。

```text
airDamageReduction
  = clamp(effectiveAA ^ 0.25 × 0.20, 0, 0.66)
```

```lua
ANTI_AIR_POW_ON_INCOMING_AIR_DAMAGE = 0.25
ANTI_AIR_MULT_ON_INCOMING_AIR_DAMAGE = 0.20
MAX_ANTI_AIR_REDUCTION_EFFECT_ON_INCOMING_AIR_DAMAGE = 0.66
```

性質は次の通り。

- 艦自身のAAは、航空機撃墜判定とダメージ軽減の両方に効く。
- 他艦のAAは、このダメージ軽減には40%係数で効く。
- 他艦のAAは、目標艦の直接撃墜数計算には入らない。
- `AA^0.25` なので、AAを積むほど増加効果は逓減する。
- 軽減率の上限は66%。

---

## 12. Strength・Organizationダメージ

`FUN_1418da3c0` で、成功した攻撃機数から基礎ダメージを計算する。

```text
H = successfulPlanes
A = 平均 naval_strike_attack（表示値ではなく0.1倍変換後の内部値）
R = airDamageReduction
```

母艦が同じ海戦に存在するだけでなく、攻撃元航空隊の発進基地を `CShip` として取得でき、その艦が現在のcombat participant groupで見つかった場合にだけ `NAVAL_STRIKE_CARRIER_MULTIPLIER` が掛かる。MOD値は1.00なので、条件を満たしても攻撃値は変化しない。

```text
A_internal  = displayedNavalAttack × 0.1 × statModifiers
A_effective = A_internal × carrierMultiplier  （発進元空母がこの海戦で見つかった場合）
A_effective = A_internal                      （それ以外）
```

ダメージ本体:

```text
rawDamage = H × A_effective × (1 - R)

strengthDamage = rawDamage × 0.8
organizationDamage = rawDamage × 1.25
```

```lua
NAVAL_STRIKE_DAMAGE_TO_STR = 0.8
NAVAL_STRIKE_DAMAGE_TO_ORG = 1.25
```

よって、同じ成功機数・同じ Naval Attack なら、OrgダメージはSTRダメージの `1.25 / 0.8 = 1.5625` 倍を基礎とする。その後、固定小数の丸め、残存Strength上限、艦・海戦側のmodifier、クリティカル処理を経て実ダメージが適用される。

設計画面で対艦攻撃26の機体を例にし、攻撃値modifierなし、成功1機、AA軽減なしとすると次になる。

```text
A_internal = 26 × 0.1 = 2.6

MODの海戦内空母攻撃:
  STR = 1 × 2.6 × 1.00 × 0.8 = 2.08

AA軽減50%なら:
  STR = 2.08 × 0.5 = 1.04
```

したがって、戦闘結果UIで1前後の航空ダメージが表示されても、設計が悪くて対艦攻撃1へ落ちたとは限らない。表示26が内部2.6へ変換され、AA軽減とSTR係数を受ければ自然に到達する値である。現在のバニラで発進元空母の参加条件を満たすなら同じ例は `2.6 × 10 × 0.8 = 20.8`（AA前）になる。

### クリティカル

爆撃関数は目標の残存Strength率とクリティカル軽減modifierからクリティカル確率を作り、`naval_fire_exchange_air.cpp:0x483` で別の乱数を振る。

成立時は `FUN_1418d4500` を通じて航空攻撃用の部位損傷判定を行う。部位損傷として処理されなかった場合には、低Strength率に応じたダメージ倍率へフォールバックする分岐がある。

最後にSTR/Orgを減らし、撃沈、攻撃航空隊へのkill credit、戦果、経験値、海戦結果UI用データを更新する。

---

## 13. 乱数が振られる場所

海戦内航空攻撃で確認できた主な乱数点は次の通り。

| ソース行識別子 | 用途 | RNG種類 |
|---:|---|---|
| `0x43d` | 目標艦の重み付き選択 | `random_int` |
| `0x27b` | 艦載戦闘機撃墜数の小数丸め | `random_fixed` |
| `0x210` | 目標艦AAが今回有効に命中するか | `random_fixed` |
| `0x21a` | 艦AA撃墜数の小数丸め | `random_fixed` |
| `0x45b` | 攻撃隊全体の爆撃成功機数 | `random_fixed` |
| `0x483` | 艦への航空クリティカル | `random_fixed` |
| `0x367` | 戦果・creditへ使う攻撃航空隊の選択 | `random_int` |

`random_fixed` の本体は `FUN_142187010 @ 0x142187010` で、0～99999の整数を返す。`random_int` は `FUN_1421871c0 @ 0x1421871c0`。

どちらもOSの乱数を毎回呼ぶのではなく、ゲーム全体の同期用シードと呼出カウンタから決定論的に生成される。ソースファイル名と行識別子も乱数ログへ記録できる構造で、マルチプレイの同期を前提としたRNGである。

---

## 14. 通常の戦略地域空戦との接続と相違点

### 共通するもの

- 同じ航空機装備stat取得系を使う。
- fighter迎撃では通常空戦と同じ agility差・speed差ヘルパーを再利用する。
- 同じ同期RNG (`random_fixed` / `random_int`) を使う。
- 基地航空隊が海戦へ到達・参加する入口は、通常の航空任務・戦略地域側と接続されている。
- 空母航空隊同士の空対空戦闘は通常空戦コアを再利用し、空母戦闘用フラグによってdamage scaleだけを3へ切り替える。

### 戦略地域側で先に起こること

海軍攻撃・港湾攻撃任務の爆撃機は、目標艦船へ到達する前の通常空戦で、同じ空域の敵戦闘機から撃墜・任務妨害を受け得る。この処理は `FUN_140beee30` → `FUN_140f45ef0` → `FUN_140f4a790` の通常空戦系であり、海戦内の `FUN_1418daad0` ではない。

```text
通常の戦略地域空戦
    基地戦闘機 vs 海軍攻撃・港湾攻撃中の爆撃機
        ↓ 撃墜・任務妨害
海戦または港湾攻撃へ到達できる爆撃機
        ↓
海戦専用strikeの場合は艦載CAP → 艦AA → 爆撃
```

そのため、「基地戦闘機は海戦内CAPへ参加しない」と「基地戦闘機は海軍攻撃中の爆撃機を目標到達前に妨害できる」は両立する。前者は海戦オブジェクト内部の参加者、後者は上流の戦略地域空戦について述べている。

### 海戦専用のもの

- 実際の攻撃解決は `naval_fire_exchange_air.cpp` の専用関数群。
- `FUN_1418db000` 以降の艦艇攻撃解決そのものは、通常の「両陣営の航空隊同士が相互に射撃する空戦ラウンド」ではない。
- 防御戦闘機迎撃 → 目標艦AA → 爆撃という一方向のstrikeパイプライン。
- 目標は艦種、screening、最大Strength、損傷率、AAから重み付き抽選。
- 艦隊全体AAによる独自の爆撃ダメージ軽減式がある。
- 空母Orgは専用の発艦効率に直接掛かる。
- 自艦空母が同じ海戦にいる場合も、このMODでは倍率1.00なのでNaval Attackは増えない。
- 外部基地航空隊には艦隊Strengthと海戦日数で決まる参加上限がある。

したがって、戦略地域の通常空戦は海戦内strikeの代替ではなく、その上流にある別段階である。通常空戦による撃墜・任務妨害は海戦へ到達できる攻撃機を減らし、海戦に参加できた機体には上記の別系統で改めて艦載CAP・AA・目標選択・爆撃処理が走る。

---

## 15. 周辺処理：航空機による潜水艦探知

海戦内航空機から潜水艦探知へ寄与させるための別define群も存在する。

```lua
NAVAL_COMBAT_AIR_SUB_DETECTION_MAX = 10.0
NAVAL_COMBAT_AIR_SUB_DETECTION_SLOPE = 10.0
NAVAL_COMBAT_AIR_SUB_DETECTION_EXTERNAL_FACTOR = 1.0
NAVAL_COMBAT_AIR_SUB_DETECTION_INTERNAL_EFFICIENCY_FACTOR = 1.0
NAVAL_COMBAT_AIR_PLANE_COUNT_TO_SUB_DETECTION = 1.0
NAVAL_COMBAT_AIR_SUB_DETECTION_DECAY_RATE = 1.0
NAVAL_COMBAT_AIR_SUB_DETECTION_FACTOR = 0.0
```

設計上は、航空機数や各種航空statから `Y × x/(x+X)` 型で探知寄与を作り、前時間からの減衰値との大きい方を採る構造になっている。

ただしMODでも最終係数 `NAVAL_COMBAT_AIR_SUB_DETECTION_FACTOR = 0.0`。したがって、この周辺システムはコードとして存在するものの、MOD設定では最終的な潜水艦探知寄与が0になる。

---

## 16. 実戦上の読み替え

内部処理をゲーム上の意味に直すと、次のようになる。

1. **空母Org低下**は主に発艦機数を直接減らす。命中率だけが落ちるのではない。
2. **基地戦闘機**は海戦内CAPには入らないが、上流の通常空戦で海軍攻撃・港湾攻撃中の爆撃機を撃墜・妨害できる。
3. **空母航空隊同士の空対空戦闘**では `COMBAT_DAMAGE_SCALE_CARRIER = 3.00` が使われ、通常空戦より双方の損害が大きくなる。
4. **艦艇攻撃直前の防御戦闘機**は爆撃前に攻撃機を減らすが、このstrike関数内では反撃損失を受けない。
5. **目標艦AA**は、直接撃墜と爆撃ダメージ軽減の二重に効く。
6. **僚艦AA**は直接撃墜には入らないが、40%係数で爆撃ダメージ軽減へ寄与する。
7. **screening崩壊**による空母の被選択重み増加は、空母のBASE/SCALEが10/10になったため発生しない。一方で輸送船は1から10へ増える。
8. **損傷艦**はさらに狙われやすい。低Strength補正は最大で基礎重みを6倍にする。
9. **爆撃成功機数は隊全体乱数1個**なので、結果のばらつきが大きい。
10. **表示上のNaval Attackは内部で0.1倍**される。さらに発進元空母が同じcombatの参加艦として見つかった場合だけ空母倍率が掛かるが、MOD値は1.00なのでバニラの10倍補正は無効化されている。
11. **大量投入**には外部航空隊参加上限がある。海戦内stacking式自体も存在するが、無補正枠10000機のため通常規模では事実上働かない。

---

## 17. 確度と注意点

- 関数名 `FUN_xxx` はシンボルのない実行ファイルへGhidraが付けた仮名。
- 各関数の意味は、呼出関係、使用するdefine、RTTI型、残存ソースパス・行番号、読書きするfieldから同定した。
- 上記の中核式、乱数点、処理順、define対応、Naval Attack／Targetingの0.1倍変換と任務ビット引渡しは高確度。
- `M_T`、carrier/country/commander modifier、クリティカル内部の細かなmodifier名は、バイナリ上では数値IDで取得されるものがあり、ここでは役割名でまとめた。
- 基地航空隊と空母航空隊の混合ペアは通常空戦コア側では排除されないが、実際に同じ空戦へ割り当てられるには戦略地域・任務側の上流条件を満たす必要がある。その全enum名とUI上の戦果帰属は未確定。
- アドレスは提供されたバイナリ専用。ゲーム更新で変わる。
- defineはMODで変更できるため、式は同じでも数値結果は導入MODにより変化する。

---

## 18. 「艦載機が時々機能しない」現象の切り分け

### 18.1 結論

確認範囲では、同一条件の艦載機を原因不明のままランダムに無効化する単一のexe不具合は見つからなかった。ただし、プレイヤーから「機能していない」と見える経路は多く、現在のMODには艦載戦闘機CAPを実質無効化する明白な設定不整合もある。

| 現象 | 判定 | 主因 |
|---|---|---|
| 夜間・吹雪で艦載攻撃機が全く出ない | 仕様 | 発艦効率の天候／夜間係数が0 |
| 豪雨・低Org時に小航空隊が出たり出なかったりする | 仕様 | 0.20、Org、甲板効率等を乗算後、整数機数へ丸める |
| 攻撃後しばらく動かない | 仕様 | MODでは成功した海戦内攻撃ごとに原則4時間待機 |
| 出撃したように見えるが艦へダメージがない | 仕様・乱数 | CAP・艦AA・攻撃成功機数の共有乱数で成功機数0になり得る |
| どの防衛スタンスでも艦上戦闘機がほぼ迎撃しない | **MOD設定不整合** | 全スタンスが攻撃1.00、防御 `1-1.00=0` |
| 海戦開始から24時間後に攻撃出撃が約20%低下する | **MOD値の不整合** | 選択値1.00から基礎値0.80へ切替 |
| Naval Strike能力・任務を持たない航空隊が艦を攻撃しない | 仕様 | 攻撃側リスト／現在任務マスクの入口で除外 |
| 空母が攻撃可能な海戦参加状態でない | 仕様だがUI対応は未同定 | 参加艦state flagのgateでstrikeを中止 |

### 18.2 攻撃が成立するまでの主なgate

海戦側の毎時更新 `FUN_141bc2d00` と1回の攻撃 `FUN_1418db000` をつなぐと、内部艦載攻撃は概ね次の順で絞られる。

```text
攻撃航空隊リストがある
  → 現在任務が Naval Strike / Port Strike / Kamikaze 系
  → 攻撃待ち時間が0
  → 発進元空母が攻撃可能な海戦参加状態
  → 発艦効率 × 空母Orgから1機以上に丸められる
  → 攻撃対象となる有効な敵艦がある
  → CAPと艦AAを通過する機体がある
  → round(残存機 × targeting係数 × 共有乱数) が1以上
  → STR／Orgダメージ
```

先頭のリスト登録では `FUN_1418d90f0` が装備の艦艇攻撃能力を確認する。再集計 `FUN_1418dca80`／`FUN_1418d83e0` では、航空隊の現在任務マスク `0x190` が海軍攻撃・港湾攻撃・神風系であることを再確認する。従って、設計上Naval Attackがあっても現在任務が該当しなければ攻撃値集計へ入らない。

また、攻撃側航空隊が0、総機数が0、敵の有効参加艦が0、攻撃性能集計が0、目標抽選に候補がない場合は早期returnする。発進元空母については、combat participantの内部state flag `+0xe0 & 0x0b` が0なら中止することも確認できる。各bitと画面上の「未到着・予備・離脱中」等との完全な対応は未同定なので、ここは単に「現在攻撃できない参加状態」とするのが安全である。

### 18.3 クールダウンと「攻撃したのに何も起きない」

内部艦載機は攻撃可能判定 `FUN_1418d98e0` で待ち時間が0のときだけ選ばれる。攻撃処理が成立すると `FUN_1418dcd60` が次回待ち時間を設定し、MOD基礎値は4時間である。毎時更新 `FUN_141bc1cb0` がこれを1ずつ減らす。

夜間や発艦0機など、実攻撃以前の早期returnでは待ち時間を消費せず、次の時間に再判定される。一方、目標を選んで後段まで進んだ後は、CAP・AAまたは次式の丸めで最終成功機数が0でも攻撃成立扱いとなり、4時間待機へ入る。

```text
successfulPlanes
  = round(remainingPlanes × pStrike × U)

U = 攻撃隊全体で1個の一様乱数
```

このため「損害表示なし→その後数時間も攻撃なし」は、航空隊が登録されていない証拠にはならない。さらにMODは `NAVAL_STRIKE_CARRIER_MULTIPLIER = 1.00` で、表示Naval Attackは内部で0.1倍されるため、命中しても1前後の小ダメージに見える場合がある。

### 18.4 MOD固有の防衛スタンス不整合

バニラとMODの値を比較する。

| 値 | バニラ | MOD v13.00 |
|---|---:|---:|
| 攻撃出撃比率5段階 | 0 / .25 / .50 / .75 / 1.00 | 1 / 1 / 1 / 1 / 1 |
| 防御出撃比率5段階 | 1 / .75 / .50 / .25 / 0 | 0 / 0 / 0 / 0 / 0 |
| 選択値の有効時間 | 24時間 | 24時間 |
| 24時間後の攻撃基礎値 | .50 | .80 |

実行コードはこの配列を飾りとして読むのではない。

- `FUN_1418d6310 @ 0x1418d6310` は選択indexから攻撃比率またはその補数を読み、発艦効率へ使う。
- `FUN_1418db000 @ 0x1418db000` は防御側について `1 - table[index]` を作り、直接CAPへ渡す。
- `FUN_1418daad0 @ 0x1418daad0` は防御戦闘機数へその比率と0.35を掛ける。
- `FUN_1418d5b00` は選択値の残り時間を24で初期化し、`FUN_141bc1cb0` が毎時減算する。

従って、現在のMODで「完全防御へ変えても艦上戦闘機が迎撃しない」なら、再現性のある説明がつく。全1.00が意図的な“常時全力攻撃”設定である可能性は残るが、5択UIと説明文を残したまま全選択を同値にしているため、少なくとも設定とUIは一致していない。

### 18.5 最短のA/B試験

原因を混ぜないため、攻撃機と戦闘機を別々に試験する。

1. 攻撃機試験は、正午・晴天・Org100%・甲板過密なし・敵戦闘機なし・低AAの単艦を使う。同じ戦闘を完全な夜間、吹雪、豪雨、Org50%で反復し、発艦機数と攻撃間隔を記録する。
2. 任務試験は同じ機体でNaval Strike系任務と非対艦任務だけを切り替える。
3. CAP試験は、防御空母へ十分な艦上戦闘機、攻撃側へ多数の艦上攻撃機を置き、目標艦AAを極小化する。現在のMODで完全防御と攻撃的を多数回比較する。予測は両方ほぼ同じで、通常のCAP撃墜がほぼ出ない。
4. 一時的にテーブルだけを `{0, .25, .50, .75, 1}` へ戻して同じCAP試験を行う。予測は完全防御ほど迎撃損失が増え、攻撃的で0へ近づく。
5. 24時間試験は、天候・時刻・Orgを固定できる状態で同一海戦を25時間以上継続し、攻撃出撃機数を比較する。MOD予測は24時間後に基礎部分だけ1.00から0.80へ下がる。

短い数回の戦果だけでは、CAP最低期待値0.001、AAの二段乱数、攻撃成功機数の共有乱数が混ざる。防衛スタンス検証は撃沈数や総ダメージではなく、十分な試行数の「CAP段階で失われた来襲機数」を比較するのが最も明確である。

横断的な不具合判定とMODファイル位置は [海空戦解析・横断メモ](../hoi4_miscellaneous_reverse_engineering_notes.md#10-空母艦載機が海戦中に機能しない現象の再検証) にも収録した。

---

## 19. 航空隊任務・艦隊任務・哨戒時の被発見側

### 19.1 結論

噂に含まれていた三つの要素は、同じ条件ではない。

| 要素 | 海戦内艦載機への影響 | 結論 |
|---|---|---|
| 航空隊の実行中任務 | 艦艇攻撃の性能集計を直接gateする | **直接影響する** |
| 空母の攻撃／防御スタンス | 攻撃出撃と直接CAPの配分を変える | **直接影響する**。ただし現MODは全選択が攻撃100% |
| 艦隊・任務部隊の任務（哨戒、打撃部隊等） | 海戦開始時に何隻が戦列へ入るかへ間接的に影響し得る | **直接の発艦許可ではない** |
| 交戦規定 | 海戦を開始・追跡・参加するかを上流で制御する | 海戦参加後の航空攻撃倍率ではない |
| 哨戒側／被発見側という陣営 | 通常遭遇の登録処理では両側とも100%で追加される | **被発見側だけ発艦不能という分岐はない** |

従って、「哨戒任務に発見された側の空母は常に飛ばない」という一般則は、対象exeの通常遭遇経路とは一致しない。一方、後から海戦へ加わる空母が未到着扱いである場合や、航空隊の実行中任務が対艦攻撃系でない場合には、結果だけを見ると同じように見える。

### 19.2 航空隊任務は艦艇攻撃へ直接効く

`CAirWing` では、少なくとも次の二段階を別々に確認している。

1. `FUN_1418d90f0` が `wing + 0x30 & 0x190` を調べ、機体・航空隊が海軍攻撃系任務を実行可能か確認する。`+0x30` は別箇所の残存assertから `GetAllowedMissionTypes()` と同定できる。
2. 攻撃直前の再集計 `FUN_1418dca80` が `wing + 0xa4 & 0x190` を調べ、その時間に実行中の任務が海軍攻撃系か確認する。

```text
eligibleForInternalCarrierStrike
  = (allowedMissionTypes & 0x190) != 0
  AND (executingMissionType & 0x190) != 0
```

`0x190` は Naval Strike、Port Strike、Kamikaze系をまとめたマスクである。二番目の値は任務別stat表を取得する関数へそのまま渡され、同関数には単一任務bitを要求するassertも残っているため、単なる設計能力マスクではなく、その時点で選ばれた実行任務として扱われている。

海戦専用コード `naval_fire_exchange_air.cpp` と `navalcombat.cpp` の確認範囲では、この `wing + 0xa4` を海戦開始時にNaval Strikeへ強制上書きする処理は見つからない。海戦側は上流の航空任務処理が決めた実行任務を読む側である。従ってUIで有効にした航空任務と、上流スケジューラがその時間に実際に選んだ任務は艦載攻撃へ影響し得る。

ただし、これは艦上戦闘機のすべての役割を止める条件ではない。艦艇攻撃直前の直接CAPは戦闘機能力を持つ航空隊リストから集計され、Naval Strike系の実行任務を要求しない。空母航空隊同士の通常dogfightは、さらに別の戦略地域空戦経路で航空任務条件を処理する。したがって次の三つは分ける必要がある。

```text
艦上攻撃機による対艦攻撃  → executingMissionType 0x190を要求
艦艇攻撃直前の艦上CAP     → 戦闘機能力と空母スタンスを使用
空母航空隊同士のdogfight  → 通常空戦側の任務割当を使用
```

### 19.3 艦隊任務は参加艦stateへ間接的に効く

空母攻撃の毎時driver `FUN_141bc2d00`、攻撃解決 `FUN_1418db000`、航空性能再集計 `FUN_1418dca80` には、`PATROL` や `STRIKE_FORCE` を直接比較して発艦を許可・禁止する分岐はない。

艦隊任務が関与する主な場所は、任務部隊を海戦参加者へ変換する `FUN_1415a37c0` である。概略は次になる。

```text
spread
  = clamp(
      MISSION_SPREADS[taskForceMission]
      + (1 - joinEfficiency) * MISSION_DEFAULT_SPREAD_BASE,
      0, 1)

initialActiveRatio
  = 1 - spread * EFFICIENCY_TO_JOIN_COMBAT_RATIO_PENALTY
```

現MODでは次が設定されている。

```text
MISSION_SPREADS[HOLD]            = 0
MISSION_SPREADS[PATROL]          = 0
MISSION_SPREADS[STRIKE_FORCE]    = 0
MISSION_SPREADS[CONVOY_RAIDING] = 0
MISSION_SPREADS[CONVOY_ESCORT]  = 0

MISSION_DEFAULT_SPREAD_BASE = 1.0
EFFICIENCY_TO_JOIN_COMBAT_RATIO_PENALTY = 1.0
```

よって通常の戦闘任務では、海戦参加時に渡された効率が1.0なら全艦が初期戦列へ入り、任務名を哨戒から打撃部隊へ変えただけでは艦載機の発艦可否は変わらない。効率が1未満の別参加経路では一部艦が戦列未到着state `4` となり得る。

空母攻撃側は発進元空母の参加stateについて `state & 0x0b != 0` を要求する。state `4` はこの判定を通らないため、当該空母が戦列へ到着するまでは内部艦載攻撃が作動しない。これが「艦隊任務によって艦載機が飛んだり飛ばなかったりする」という観察へつながる、実装上もっとも近い経路である。ただし原因は任務名そのものではなく、その任務・参加経路・参加効率から作られた艦の初期参加stateである。

### 19.4 通常の哨戒遭遇では被発見側も100%参加

通常の接触から海戦を作る `FUN_140b93cd0 @ 0x140b93cd0` は、二つの任務部隊を次のように追加する。

```text
addSideA(combat, taskForceA, 100000)
addSideB(combat, taskForceB, DAT_14288f758)
```

`DAT_14288f758 @ 0x14288f758` の実値はlittle endianで `100000`、すなわち1.0である。片方だけ低い効率を渡しているように見えるデコンパイル表示だが、実際には両側とも同じ100%である。

哨戒の `MISSION_SPREADS[PATROL] = 0` なので、この通常遭遇では次になる。

```text
spread = 0 + (1 - 1) * 1 = 0
initialActiveRatio = 1
```

従って、哨戒側と被発見側のどちらも全艦が初期アクティブ候補であり、空母も他の条件を満たせば攻撃可能stateになる。参加登録wrapperは攻撃側・防御側の別コレクションへ艦を入れるが、海戦内航空driverは各陣営の航空交換groupに対して同じ関数を呼ぶ。`FUN_141bc2d00` にも「攻撃側だけ」「発見した側だけ」を許す条件はない。

以上から、少なくともこの通常遭遇経路については次の噂を否定できる。

```text
誤: 哨戒で発見された側の艦載機は仕様上発艦しない
正: 両側とも100%で海戦へ登録され、航空処理も同じ関数で行われる
```

### 19.5 噂と同じ見え方を作る条件

被発見側の航空戦果がない場合、優先して切り分ける順は次の通り。

1. **その空母が初期遭遇艦隊ではなく増援・打撃部隊として後から参加していないか。** 後参加経路では低い参加効率が渡され、空母がstate `4` になり得る。
2. **艦上攻撃機の実行中任務が `0x190` に入っているか。** 機体にNaval Attackがあっても、実行任務が別なら対艦性能を集計しない。
3. **夜間、吹雪、低Org、甲板効率、整数丸めで発艦数0になっていないか。**
4. **直前の攻撃後4時間待機中ではないか。**
5. **CAP・AA・共有乱数で有効命中0になっただけではないか。**
6. **「艦上攻撃機の対艦攻撃」「直接CAP」「通常dogfight」のどの戦果を見ているか。** 三経路は独立している。

とくに「哨戒艦隊が発見し、別基地の打撃部隊が到着する」場面では、発見側／被発見側の違いと、初期参加／後参加の違いが混同されやすい。コード上で発艦停止へつながるのは後者である。

### 19.6 残る留保

- 通常遭遇 `FUN_140b93cd0` 以外には、一方へ可変の参加効率を渡して既存海戦へ合流させる経路がある。全種類の遭遇・増援について常に左右100%とはいえない。
- 海戦専用モジュール内に任務の強制上書きはないが、その直前の戦略航空スケジューラが空母航空隊の実行任務を選ぶ全条件までは未同定である。従って「UIで一つの任務を押した瞬間に必ず `+0xa4` がそのbitになる」とまでは断定しない。
- ただし海戦内攻撃の時点で `+0xa4 & 0x190` を実際に要求することと、被発見側専用の無効化条件がないことは高確度である。
