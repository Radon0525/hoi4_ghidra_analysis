# HOI4 戦略地域から艦隊へ行う通常対艦攻撃の内部処理

> **innovative balance v13.00 対応版**  
> 係数は Workshop ID `820039020` の `common/defines/00_defines.lua` を基準にしている。実行ファイル側の処理構造・関数アドレスは元のGhidra解析結果、数値はMOD値である。

## 0. 調査対象と結論

この文書は、戦略地域に存在する通常の艦隊へ航空隊が `Naval Strike` を行う場合を対象とする。典型例は陸上基地の攻撃機だが、空母を発進基地とする航空隊が戦略地域任務として外部参加する場合も、入口は同じ系統である。すでに海戦が発生している場所へ参加する場合との違いも後半で明示する。

最も重要な結論は次の通り。

1. 通常の対艦任務は、このMODでは各戦略地域について **12時間に1回**、地域ごとにずらされた時刻に処理候補になる。
2. その時刻になっても必ず爆撃できるわけではない。地域内の航空戦、実働機数、敵艦隊の発見判定、攻撃可能条件を順に通過する必要がある。
3. 発見された海戦外の艦隊は、1回の攻撃を処理するためだけに **一時的な `CCombatNaval`** へ入れられる。
4. この一時海戦は直ちに解決・破棄されるが、目標艦抽選、艦載戦闘機の迎撃、艦AA、命中、ダメージ、クリティカル、撃沈といった中核部分には、実際の海戦と同じ `naval_fire_exchange_air.cpp` の処理が使われる。
5. したがって、海戦外の Naval Strike は「簡略化された別の直接ダメージ式」ではない。**入口と寿命は別だが、航空攻撃の中核計算は海戦用のものを再利用する**仕組みである。
6. ただし空母発進時の10倍補正は「同じ関数を通る」だけでは発動しない。発進元空母が、その攻撃に使う同じ `CCombatNaval` の参加艦として見つかることが追加条件である。

以下では、内部の固定小数 `100000 = 1.0` を通常の実数へ直して表記する。define の値は `innovative balance` v13.00 の設定値であり、MOD更新によって変更されうる。

---

## 1. 処理全体の流れ

通常の基地航空隊による対艦攻撃は、概ね次の順番で進む。

```text
戦略地域の航空更新時刻を確認
  ↓ その地域の12時間周期に該当
地域内の通常空戦を処理
  ↓ 撃墜・妨害されず、任務に使える機体を算出
Naval Strike 任務を振り分ける
  ↓
地域内の海戦・艦隊・移動中部隊を列挙
  ↓ 海戦外の艦隊には発見判定
攻撃対象候補を作る
  ↓
既存海戦なら、その海戦へ外部航空隊として登録
海戦外艦隊なら、一時的な海戦オブジェクトを生成
  ↓
投入可能機数を制限
  ↓
目標艦抽選 → 艦載戦闘機CAP → 目標艦AA
  ↓
攻撃成功機数抽選 → 艦隊AA軽減 → STR/Org損害
  ↓
撃墜・戦果・経験値などを反映
  ↓
一時海戦を直ちに破棄
```

主な関数は次の通り。

| 役割 | 関数 |
|---|---:|
| 戦略地域の処理位相を設定 | `FUN_140deb2a0` |
| 現在時刻が地域の処理時刻か判定 | `FUN_140debc20` |
| 航空任務の振り分け | `FUN_140f46600 @ 0x140f46600` |
| Naval Strike の対象列挙・発見 | `FUN_140f41a40 @ 0x140f41a40` |
| 地域側の航空探知効率 | `FUN_140f3bc90` |
| 任務に使える実働機数 | `FUN_140f3c8b0` |
| 作られた海軍目標を実行 | `FUN_140f47190` |
| 通常艦隊用の一時海戦 | `FUN_140b92390 @ 0x140b92390` |
| 移動中部隊用の一時海戦 | `FUN_140b92510 @ 0x140b92510` |
| 一時海戦の解決 | `FUN_14154b340 @ 0x14154b340` |
| 海戦の航空攻撃ドライバ | `FUN_141bc2d00 @ 0x141bc2d00` |
| 1回の航空攻撃本体 | `FUN_1418db000 @ 0x1418db000` |

---

## 2. 「数時間に一回」の正体

### 2.1 MODでは12時間周期

関連 define は次の値である。

```lua
NAir.HOURS_DELAY_AFTER_EACH_COMBAT = 6
```

ただし実装はこの値をそのまま4時間周期として使わず、往復時間の抽象化として2倍する。

```text
周期 = HOURS_DELAY_AFTER_EACH_COMBAT × 2
     = 6 × 2
     = 12時間
```

define のコメントにも、一般に往復のため2倍して使われる旨が記されている。

### 2.2 全地域が同じ時刻に動くわけではない

`FUN_140deb2a0` は各戦略地域に位相を設定する。

```text
regionPhase = regionId mod (HOURS_DELAY_AFTER_EACH_COMBAT × 2)
```

`FUN_140debc20` は、現在時刻について次を判定する。

```text
処理する ⇔
regionPhase == currentHour mod (HOURS_DELAY_AFTER_EACH_COMBAT × 2)
```

したがって、このMODでは各地域が12時間ごとに更新される一方、地域IDによって処理時刻が分散されている。全世界の航空戦を同じ時間に集中させないための構造と考えられる。

重要なのは、これは「必ず艦隊に爆弾が命中する間隔」ではなく、**その地域の任務解決を試みる時刻**だという点である。対象艦隊を発見できなかった、攻撃機が妨害された、実働機が0だった、攻撃対象が無効だった、といった場合には何も起きない。

### 2.3 港湾攻撃とは別

港湾攻撃は任務ビットも対象収集関数も別であり、さらに次の define を持つ。

```lua
NAir.PORT_STRIKES_DELAY_MULTIPLIER = 2
```

この文書の12時間周期と処理経路は、海上にいる艦隊への通常 `Naval Strike` を中心に述べたものである。港湾攻撃の `PORT_STRIKES_DELAY_MULTIPLIER = 2` はMODでも維持されているが、港湾攻撃の周期・機数制限を、そのまま移動中艦隊への攻撃へ流用してはいけない。

---

## 3. 艦隊攻撃の前に地域空戦がある

基地から飛来する対艦攻撃機は、いきなり艦船に対してダメージ判定を行うわけではない。戦略地域側では通常の航空戦処理が先に存在し、その地域で迎撃任務を行う敵戦闘機は、攻撃機を撃墜または妨害できる。

追跡できた大きな経路は次の通り。

```text
strategicair.cpp 側の地域航空戦
  FUN_140beee30
    ↓
airmission.cpp 側の通常空戦
  FUN_140f45ef0
    ↓
撃墜・妨害計算
  FUN_140f4a790
    ↓
生き残り、任務を遂行できる機体
    ↓
海軍目標の収集と艦艇攻撃
```

この層と、後述する防御艦隊の艦載戦闘機CAPは別物である。

- 基地の敵戦闘機: 戦略地域の通常空戦で攻撃機を妨害・撃墜する。
- 目標艦隊の艦上戦闘機: 一時海戦の `naval_fire_exchange_air.cpp` 内で、生き残った攻撃機を艦艇到達直前に迎撃する。

つまり、条件がそろえば対艦攻撃機は「地域空戦」と「目標艦隊の艦載CAP」という二段階の防空を受けうる。

---

## 4. Naval Strike 任務の入口

`FUN_140f46600` は航空任務の種類をビットで振り分ける。今回重要な分岐は次の通り。

| 任務 | ビット | 経路 |
|---|---:|---|
| Naval Strike | `0x10` | 海軍目標を収集して攻撃 |
| Kamikaze Strike | `0x80` | 海軍目標を収集して特殊攻撃 |
| Port Strike | `0x100` | 港湾用の別対象収集へ分岐 |

通常の Naval Strike と特攻は `FUN_140f41a40` で地域内の海軍目標を集め、候補を作った後、`FUN_140f47190` で実行する。港湾攻撃はここで別経路へ分かれる。

### 4.1 任務に使える機数

入口では `FUN_140f3c8b0` が、航空隊の現在機数、任務効率・参加効率などから、今回の任務に実際に使える機数を固定小数で計算し、丸め・上限制限を行う。

概念的には次の形である。

```text
effectivePlanes ≈ currentPlanes × missionParticipationEfficiency
```

実際には機種・任務適合性を含む補正と固定小数の丸めが挟まる。ここで0機なら艦隊探索へ進まない。

また、この値は最終投入機数ではない。後で一時海戦側の external-plane cap によってさらに制限される。

---

## 5. 地域内の艦隊をどう発見するか

### 5.1 既存海戦と海戦外艦隊は扱いが違う

`FUN_140f41a40` は、戦略地域内の州・海軍部隊・海戦を列挙する。

- すでに発生中の海戦: 敵対関係などの条件を満たせば候補になる。戦闘中なので、通常の海戦外艦隊に使う同じ発見抽選は通さない。
- 海戦外の通常艦隊: 活動状態、位置、敵対関係などを確認した後、艦隊ごとに発見抽選を行う。
- 移動中・輸送系の海軍部隊: 専用補助関数を通るが、同種の可視性・探知抽選を持つ。

### 5.2 通常艦隊の発見確率

海戦外艦隊については、艦隊の水上可視性と潜水艦可視性を別々に求め、大きい方を使う。

```text
V = max(V_surface, V_sub)
```

そのうえで、通常の Naval Strike では概ね次の確率で候補へ入る。

```text
P_detect = clamp(
    V
  × NAVAL_STRIKE_DETECTION_BALANCE_FACTOR
  × regionalAirDetectionEfficiency,
  0, 1
)
```

MOD値は次の通り。ここはバニラから変更されていない。

```lua
NAir.NAVAL_STRIKE_DETECTION_BALANCE_FACTOR = 0.5
```

したがって、実数化した簡略式は次になる。

```text
P_detect = clamp(V × 0.5 × D_region, 0, 1)
```

ここで `D_region` は攻撃側国家の、その戦略地域における航空探知効率である。単なる航空任務効率ではない。`FUN_140f3bc90` がレーダー、地域内航空機、夜間、国家補正などを含む `AIR_DETECTION_IN_REGION_DETAILS` 相当の値を作っている。

実装では、同期乱数から `R = 0..99999` を作り、次を比較する。

```text
発見成功 ⇔ V × 0.5 × D_region ≥ R / 100000
```

### 5.3 艦隊可視性の組み立て

水上・潜水艦可視性は `FUN_140d17330` と `FUN_140d175e0` で求められる。概念式は次の通り。

```text
V_surface = max(0,
    taskForceSurfaceVisibility
  × VISIBILITY_MULTIPLIER_FOR_SPOTTING
  × returnForRepairMultiplier
  × (1 + navalVisibilityModifier)
)

V_sub = max(0,
    taskForceSubVisibility
  × VISIBILITY_MULTIPLIER_FOR_SPOTTING
  × returnForRepairMultiplier
  × (1 + navalVisibilityModifier)
)
```

確認できたMOD値は次の通り。ここはバニラから変更されていない。

```lua
NNavy.VISIBILITY_MULTIPLIER_FOR_SPOTTING = 0.1
NNavy.NAVY_VISIBILITY_BONUS_ON_RETURN_FOR_REPAIR = 0.9
```

修理帰投状態なら `returnForRepairMultiplier = 0.9`、それ以外は `1.0` となる。変数名上は bonus だが、この値をそのまま乗算するため、MODでも修理帰投中の可視性項が10%下がる形になる。

### 5.4 数値例

仮に最終的な `V = 0.8`、地域航空探知効率 `D_region = 0.7` なら、

```text
P_detect = 0.8 × 0.5 × 0.7
         = 0.28
```

この艦隊は、その回の地域処理で28%の確率で攻撃候補になる。この抽選は12時間周期の実行機会ごとに行われるため、発見できない回が続けば爆撃間隔は12時間より長くなる。

---

## 6. 発見後に作られる「一時海戦」

### 6.1 通常艦隊の場合

海戦外の艦隊が攻撃対象になると、`FUN_140f47190` から `FUN_140b92390` が呼ばれる。この関数はローカル領域に完全な `CCombatNaval` を構築する。

確認できた処理順は次の通り。

```text
FUN_141549f40(localCombat)       CCombatNavalを構築
localCombat.strikeOnlyFlag = 1  一回限りの攻撃用フラグ
FUN_14154a5c0(localCombat)       両陣営のCNavalCombatantを作る
FUN_14154cc60(localCombat, seaProvince)
FUN_1413802b0(localCombat, targetTaskForce, ...)
FUN_1415a7240(targetSide)        fire-exchange用グループを準備
FUN_14154c710(localCombat)       初期化
targetSide.vfunc+0xf0(
    attackingAirWing,
    remainingPlanes,
    missionCoefficient,
    0
)                                  外部航空隊を登録
FUN_14154b340(localCombat, resultBuffer)
FUN_140b912e0(resultBuffer)      戦果・経験値などを反映
FUN_14154b3d0(localCombat)       後処理
FUN_14154c6e0(localCombat)       終了処理
FUN_14154a110(localCombat)       即座に破棄
```

この海戦はマップ上で持続する通常海戦として登録されるわけではない。1回の航空攻撃を共通海戦コードで解決するための入れ物であり、処理の最後に消える。

重要なのは、ここへ実艦として追加されるのは目標側の `targetTaskForce` であり、攻撃航空隊の発進元空母ではないことである。攻撃航空隊はあくまでexternal planesとして登録される。この構成が、後述する空母Naval Attack倍率の条件に影響する。

### 6.2 移動中・輸送系部隊の場合

`FUN_140b92510` が同様の一時海戦を作り、`FUN_14154bb60` で `CNavalUnitTransfer` を参加させる。その後の航空隊登録、解決、結果反映、破棄は通常艦隊とほぼ同じである。

### 6.3 すでに海戦中の艦隊の場合

対象が既存の海戦なら一時海戦は作らない。攻撃側国家から見て敵側の `CNavalCombatant` を選び、その陣営の仮想関数 `+0xf0`、実体 `FUN_1415a9410` を通じて、基地航空隊をその実在海戦へ external planes として登録する。

この違いは後述する機数上限に影響する。

---

## 7. 実際に艦隊へ突入できる機数

### 7.1 external-plane cap

基地航空隊は海戦コード上で external planes として扱われる。参加可能機数には次の上限がある。

```text
battleDays = floor(combatHours / 24)

externalCap = max(
    NAVAL_COMBAT_EXTERNAL_PLANES_MIN_CAP,
    trunc(
        totalShipCurrentStrength
      × NAVAL_COMBAT_EXTERNAL_PLANES_JOIN_RATIO
      × (1 + battleDays
             × NAVAL_COMBAT_EXTERNAL_PLANES_JOIN_RATIO_PER_DAY)
    )
)
```

MOD値は次の通り。

```lua
NNavy.NAVAL_COMBAT_EXTERNAL_PLANES_JOIN_RATIO = 0.025
NNavy.NAVAL_COMBAT_EXTERNAL_PLANES_JOIN_RATIO_PER_DAY = 0.0
NNavy.NAVAL_COMBAT_EXTERNAL_PLANES_MIN_CAP = 20
```

すでに外部航空隊が参加している通常海戦では、さらに既参加機数を差し引く。

```text
availableCap = externalCap - alreadyAssignedExternalPlanes
actualJoined = min(requestedEffectivePlanes, availableCap)
```

### 7.2 海戦外への単発攻撃ではどうなるか

一時海戦は作られたばかりなので `combatHours = 0`、かつ既参加の外部航空隊も存在しない。したがって通常の海戦外艦隊への1回の攻撃では、式は次まで簡略化できる。

```text
externalCap = max(
    20,
    trunc(totalShipCurrentStrength × 0.025)
)

actualJoined = min(effectivePlanes, externalCap)
```

例:

- 艦隊の現在総Strengthが200なら、`200 × 0.025 = 5` なので最低保証により20機。
- 現在総Strengthが1000なら25機。
- 現在総Strengthが3000なら75機。

MODでは日数係数そのものが0なので、通常の持続海戦でも海戦継続日数による上限増加はない。一時海戦も毎回破棄され、次の12時間周期では新しい一時海戦が作られる。

### 7.3 `NAVAL_STRIKE_BASE_STR_TO_PLANES_RATIO` との区別

```lua
NAir.NAVAL_STRIKE_BASE_STR_TO_PLANES_RATIO = 0.05
```

という紛らわしい define もある。しかし、この値の実行時参照は港湾攻撃の爆撃機上限計算 `FUN_140bda5c0` にあり、海上を移動中の通常艦隊に対する上記 external-plane cap とは別である。

---

## 8. 一時海戦内で使われる共通航空攻撃処理

`FUN_14154b340` から各海戦陣営のフェーズが呼ばれ、最終的に fire-exchange group の `FUN_141bc1cb0`、航空攻撃ドライバ `FUN_141bc2d00`、1回の航空攻撃本体 `FUN_1418db000` へ到達する。

攻撃本体の順番は次の通り。

1. 投入機の性能を集計する。
2. 機数過密による効率低下を計算する。
3. 重み付き乱数で目標艦を1隻選ぶ。
4. 防御艦隊に空母戦闘機がいればCAP迎撃を行う。
5. 選択された目標艦自身のAAで攻撃機を撃墜する。
6. 生き残った機数に対し、今回の攻撃成功機数を乱数で決める。
7. 目標艦AAと艦隊AAからダメージ軽減率を求める。
8. 目標艦へStrength・Organizationダメージを与える。
9. クリティカル、撃沈、戦果、航空隊損失を反映する。

以下の式は既存海戦内の航空攻撃と共通である。

### 8.1 任務別statの選択と表示値の0.1倍変換

攻撃性能を集計する `FUN_1418d83e0` は、通常Naval Strike／Kamikazeなら任務マスク `0x90`、Port Strikeなら `0x100` で航空隊を絞る。そのうえで航空隊の `wing+0xa4` に保存された実際の任務ビットを、Naval Attack getter `FUN_140f26580` とNaval Targeting getter `FUN_140f26910` へ渡す。

したがって、Naval Strikeとして収集された航空隊が、対艦性能を読む瞬間だけ「任務なし」のstatを参照する通常経路にはなっていない。設計画面で対艦攻撃が `1 + 25 = 26` と表示されているNaval Strike機は、その任務固有+25を含む26から計算される。

ただし、getterは表示値をそのまま返さず0.1倍する。

```text
A_internal = displayedNavalAttack    × 0.1 × attackStatModifiers
T_internal = displayedNavalTargeting × 0.1 × targetingStatModifiers
```

設計画面の対艦攻撃26、対艦照準16.5は、追加modifierを無視すれば内部値2.6、1.65になる。後続の命中・ダメージ式で使う `A` と `T` はこの内部値である。

---

## 9. 機数過密

目標艦隊側の艦種構成から、航空攻撃を受けられる見かけ上の容量 `L` を作る。

```text
L = max(10000, targetFleetSilhouettePlaneCapacity)
```

航空攻撃に参加する機数を `N` とすると、過密効率は次になる。

```text
stackingEfficiency = 1                         (N ≤ L)

stackingEfficiency = 1 / (1 + (N-L) × 0.005) (N > L)
```

容量へ寄与する艦種別係数はMODでも概ね次の通り。

| 艦種 | 容量係数 |
|---|---:|
| 主力艦 | 10 |
| 直衛艦 | 5 |
| 空母 | 16 |
| 支援系 | 3 |
| 輸送船 | 4 |
| 潜水艦 | 7 |

直衛艦には容量を減らす別係数 `0.02` もある。ただしMODでは固定下限が10000機なので、通常規模の海戦外攻撃では事実上stackingペナルティは発生しない。10000機を超える場合だけ、参加後の共通strike側でこの過密計算が効く。

---

## 10. どの艦が狙われるか

目標艦は均等抽選ではない。各艦に重みを付けて、`FUN_1418d9e40` が重み付き乱数で1隻を選ぶ。

艦 `i` の概念式は次の通り。

```text
classFactor_i = E × BASE[class_i]
              + (1-E) × SCALE[class_i]

weight_i = MaxStrength_i
         × classFactor_i
         × (1 + (1-StrengthRatio_i) × 5)
         + clamp(5-AA_i, 0, 5) × 12.5
```

`E` は防御側のscreening efficiencyである。艦種別のMOD値は次の通り。

| 艦種 | BASE | SCALE |
|---|---:|---:|
| 潜水艦 | 10 | 10 |
| 直衛艦 | 10 | 10 |
| 主力艦 | 10 | 50 |
| 空母 | 10 | 10 |
| 輸送船 | 1 | 10 |

式から読み取れる性質は次の通り。

- 最大Strengthが大きい艦ほど重くなる。
- すでに損傷している艦ほど、`(1-StrengthRatio) × 5` により重くなる。
- AAが5未満の艦は、MODの `NAVAL_COMBAT_AIR_LOW_AA_TARGET_SCORE = 12.5` により追加で狙われやすくなる。
- screeningが崩れると主力艦の係数は10から50へ増える。空母は10のままなので、バニラのような空母への特別な増加はない。輸送船は1から10へ増える。

艦隊を発見する乱数と、艦隊内の1隻を選ぶ乱数は別である。

---

## 11. 防御側空母の艦上戦闘機による迎撃

海戦外の艦隊でも、一時海戦へ実際の目標task forceが参加する。そのため対象艦隊に空母と艦上戦闘機があれば、`FUN_1418daad0` のCAP迎撃が機能する。

爆撃機数を `B`、防御側艦上戦闘機数を `F` とすると、概ね次の順で有効戦闘機数を作る。

```text
F_defend  = F × 0.35
F_engaged = min(F_defend, B × 1.0)
```

基礎的な迎撃力は次の形である。

```text
D0 = F_engaged × 0.30 × fighterAirAttack

D = D0
  + speedBonus
  - agilityPenalty
```

その後、攻撃機の航空防御で割って期待撃墜数を作る。

```text
expectedKills = max(
    0.001,
    0.01 × D / bomberAirDefence
)
```

期待値の整数部分はそのまま、端数は別の同期乱数で確率的に切り上げられる。

ここで働くのは **攻撃対象艦隊が持つ艦上戦闘機** である。近隣航空基地の防御戦闘機をこの一時海戦へ直接入れる処理ではない。基地戦闘機は前述の戦略地域側通常空戦で働く。

---

## 12. 目標艦自身のAAによる撃墜

艦上戦闘機迎撃後に残った攻撃機へ、選ばれた目標艦自身のAAが直接射撃する。まず攻撃機の平均Naval Strike Agilityから交戦確率を調整する。

```text
qAdjusted = max(
    0.01,
    0.7 + 0.5 × (0.7-attackerAvgNavalStrikeAgility)
)

aaEngageChance = qAdjusted × 0.65
```

この確率ロールに成功した場合、目標艦AAから撃墜率を作る。

```text
aaKillFraction = clamp(targetShipAA × 0.0006, 0, 1)

expectedAAKills = remainingPlanes × aaKillFraction
```

ここでも端数は同期乱数で丸められる。

この「目標艦AAによる機体撃墜」と、後述する「艦隊全体AAによる爆撃ダメージ軽減」は別判定である。

---

## 13. 攻撃成功機数の抽選

艦載CAPと目標艦AAを生き残った機数を `N_survive` とする。攻撃側の平均 Naval Strike Targeting（表示値を0.1倍した内部値）と補正から、成功確率の上限項を作る。

```text
pStrike = clamp(
    avgNavalStrikeTargeting
  × targetingModifiers
  × 0.30,
  0, 1
)
```

ただし、実装は各機が独立に `pStrike` で命中判定する二項分布ではない。攻撃グループ全体に対して一つの一様乱数 `U ∈ [0,1)` を引き、次で成功機数を決める。

```text
successfulPlanes = round(N_survive × pStrike × U)
```

このため、同じ機数・性能でも結果の振れが大きい。期待値だけを見れば、丸めを無視して次に近い。

```text
E[successfulPlanes] ≈ N_survive × pStrike / 2
```

「Naval Targeting × 0.30 が、そのまま各機の独立命中率」と解釈すると実装とずれる点に注意が必要である。

---

## 14. 艦隊AAによるダメージ軽減

命中側の成功機数が決まった後、目標艦自身だけでなく艦隊全体のAAも爆撃ダメージを軽減する。

```text
fleetAAComponent = fleetAAEfficiency
                 × Σ(participatingShipsAA)
                 × 0.40

effectiveAA = targetShipAA + fleetAAComponent

damageReduction = clamp(
    effectiveAA^0.25 × 0.20,
    0,
    0.66
)
```

したがって、艦隊AAによる軽減率は単純比例ではない。

- AAが増えるほど軽減率は上がる。
- 指数が `0.25` なので限界効用は逓減する。
- 最終的な軽減率には66%の上限がある。
- 目標艦自身のAAは、機体撃墜判定とダメージ軽減の両方に関係する。

---

## 15. Strength・Organizationダメージ

海戦外艦隊攻撃では、表示値を0.1倍した内部Naval Strike Attackへ各補正を掛けた値を `A_eff` とすると、概念的な生ダメージは次の通り。

```text
A_internal = displayedNavalAttack × 0.1 × statModifiers
A_eff      = A_internal × (sourceCarrierFoundInThisCombat
                         ? carrierMultiplier : 1)

rawDamage = successfulPlanes
          × A_eff
          × (1-damageReduction)
```

これにStrengthとOrganizationそれぞれの係数を適用する。

```text
StrengthDamage     = rawDamage × 0.8
OrganizationDamage = rawDamage × 1.25
```

その後、固定小数の丸め、残存Strengthによる上限、各種修正、クリティカル、撃沈、戦果記録が適用される。

### 15.1 空母発艦倍率の厳密な発動条件

```lua
NAir.NAVAL_STRIKE_CARRIER_MULTIPLIER = 1.00
```

`FUN_1418db000` は攻撃航空隊の発進基地を `CShip` として取得し、その艦を現在のcombat participant groupから `FUN_141bc04e0` で検索する。発進元空母が**この同じ海戦の参加艦として見つかった場合だけ**、攻撃値へ `NAVAL_STRIKE_CARRIER_MULTIPLIER` を掛ける。

経路別には次のようになる。

| 攻撃経路 | 発進元空母を同じcombatで検索できるか | 空母倍率 |
|---|---|---|
| 陸上基地から海戦外艦隊へ | 発進元が艦ではない | なし |
| 空母から海戦外艦隊へ戦略地域任務 | 一時海戦には通常、目標艦隊だけが実艦参加 | 通常はなし |
| 陸上基地から既存海戦へ | 発進元が艦ではない | なし |
| その既存海戦に実際に参加中の空母から攻撃 | 発進元空母が同じ海戦で見つかりうる | あり |

現在のバニラ値は10.0だが、innovative balance v13.00は1.00である。そのためMODでは、仮に参加条件を満たしても追加ダメージ倍率は付かない。

この点により、「海戦外Naval Strikeも海戦内と同じダメージ関数を使う」と「海戦外Naval Strikeには通常、空母10倍が掛からない」は両立する。同じ関数内の条件分岐へ到達するものの、一時海戦の参加艦構成では条件が偽になるためである。

同様に、基地航空隊へ空母の発艦効率を適用するわけでもない。目標側の空母は防御艦載CAPを提供しうるが、攻撃側基地航空隊が空母発艦機として扱われるわけではない。

### 15.2 対艦攻撃26が約1ダメージになりうる理由

成功1機、攻撃値modifierなし、AA軽減50%の例では次になる。

```text
A_internal = 26 × 0.1 = 2.6
StrengthDamage = 1 × 2.6 × (1-0.50) × 0.8
               = 1.04
```

したがって戦闘結果UIの約1ダメージは、「任務固有+25が無視されて対艦攻撃1になった」ことを示す証拠ではない。26を正しく読んだうえで、内部0.1倍、AA軽減、STR係数0.8を通した値だけで説明できる。

---

## 16. 乱数が振られる場所

この処理には複数の独立した乱数がある。いずれもマルチプレイ同期を前提としたゲーム内の決定的乱数列であり、各PCがOS乱数を勝手に引くような仕組みではない。

| 段階 | 乱数の用途 | 追跡位置・性質 |
|---|---|---|
| 艦隊発見 | 可視性×地域航空探知との比較 | `airmission.cpp` 側、`0..99999` |
| 一時海戦のグループ対応 | 対応候補が複数ある場合の選択 | `combatnaval.cpp` line-id `0x83` |
| 目標艦選択 | 艦ごとのweightを使う重み付き抽選 | `naval_fire_exchange_air.cpp` line-id `0x43d` |
| 艦載CAP | 期待撃墜数の端数丸め | line-id `0x27b` |
| 目標艦AA | AAが攻撃機と交戦するか | line-id `0x210` |
| 目標艦AA | 期待撃墜数の端数丸め | line-id `0x21a` |
| 攻撃成功機数 | グループ全体で共有する `U` | line-id `0x45b` |
| クリティカル | クリティカル発生判定 | line-id `0x483` |
| 戦果帰属 | どの航空隊へ戦果を付けるか | line-id `0x367` |

処理時刻そのものは乱数ではない。地域IDから決めた位相と現在時刻の剰余で決定される。ランダムなのは、その時刻に艦隊を発見できるか、どの艦を狙うか、何機が撃墜されるか、何機が攻撃に成功するか、といった各解決段階である。

---

## 17. 擬似コードでまとめた全処理

実装を単純化すると、通常の海戦外 Naval Strike は次のように表現できる。

```text
if currentHour % (HOURS_DELAY_AFTER_EACH_COMBAT*2) != regionPhase:
    return

resolveRegionalAirCombat()

planes = calculateEffectiveMissionPlanes(airWing)
if planes <= 0:
    return

for each enemyTaskForce in strategicRegion:
    if not operationallyValid(enemyTaskForce):
        continue

    Vsurface = calculateSurfaceVisibility(enemyTaskForce)
    Vsub     = calculateSubVisibility(enemyTaskForce)
    V        = max(Vsurface, Vsub)
    D        = calculateRegionalAirDetection(attackerCountry, region)

    if random01() > clamp(V * 0.5 * D, 0, 1):
        continue

    combat = createTemporaryNavalCombat()
    combat.strikeOnly = true
    combat.addDefendingTaskForce(enemyTaskForce)

    cap = max(20, trunc(enemyTaskForce.currentTotalStrength * 0.025))
    joinedPlanes = min(planes, cap)
    combat.addExternalAttackingAirWing(airWing, joinedPlanes)

    targetShip = weightedRandomShip(enemyTaskForce)

    joinedPlanes -= defendingCarrierFighterKills(
        enemyTaskForce, airWing, joinedPlanes
    )

    joinedPlanes -= targetShipAAKills(
        targetShip, airWing, joinedPlanes
    )

    A = displayedNavalAttackForCurrentMission(airWing) * 0.1
    T = displayedNavalTargetingForCurrentMission(airWing) * 0.1

    pStrike = clamp(
        T
        * targetingModifiers
        * 0.30,
        0, 1
    )

    successfulPlanes = round(joinedPlanes * pStrike * random01())

    reduction = fleetAADamageReduction(targetShip, enemyTaskForce)
    rawDamage = successfulPlanes
              * A
              * (1-reduction)

    applyStrengthDamage(targetShip, rawDamage * 0.8)
    applyOrganizationDamage(targetShip, rawDamage * 1.25)
    resolveCriticalSinkingCreditAndXP()

    destroyTemporaryNavalCombat(combat)
```

実際のコードには候補リスト、残り機数、複数航空隊、移動中部隊、固定小数丸め、国家・提督・装備modifierなどが加わるが、中心構造はこの通りである。

---

## 18. 海戦外攻撃・既存海戦・港湾攻撃の違い

| 項目 | 海戦外の通常艦隊 | 既存海戦への航空参加 | 港湾攻撃 |
|---|---|---|---|
| 海軍combat | 一時的に生成し即破棄 | 既存の持続海戦を使用 | 港湾用入口から海軍攻撃処理へ接続 |
| 通常の艦隊発見抽選 | あり | 戦闘中なので同じ抽選はなし | 港・州側の別対象選定 |
| 外部機上限 | 毎回既参加0機 | 日数では増えず、既参加機を差引く | 港湾専用機数上限あり |
| 砲撃・追跡・増援 | 持続しない | 通常海戦として持続 | 通常艦隊戦ではない |
| 地域基地戦闘機 | 上流の通常空戦 | 上流の通常空戦 | 上流の通常空戦 |
| 防御艦載戦闘機CAP | 目標艦隊にいれば作動 | 防御側空母にいれば作動 | 対象構成に依存 |
| 艦への航空攻撃式 | 共通strike式 | 共通strike式 | 海軍strike側の共通部分を利用 |
| 発進元空母の攻撃倍率 | 通常は不成立。一時海戦へ発進元空母を追加しない | 発進元空母がその海戦の参加艦なら成立 | 対象構成・入口に依存 |

海戦外攻撃は「通常海戦を始める」のではない。通常海戦のデータ構造と航空攻撃解決器を、一発分だけ借りていると表現するのが正確である。

---

## 19. プレイヤー視点で起きる現象の説明

### 爆撃がちょうど12時間おきにならない理由

処理機会は12時間周期だが、毎回すべてを通過するとは限らない。

- 敵戦闘機による撃墜・妨害
- 任務効率低下による実働機不足
- 艦隊発見抽選の失敗
- 艦隊の活動状態・位置・敵対条件
- external-plane cap
- 艦載戦闘機と艦AAによる撃墜
- グループ共有乱数による攻撃成功機数の大振れ

このため、戦闘ログ上は12時間、24時間、36時間以上と間が空くことがあり、同じ条件に見えても損害は大きく変動する。

### 大航空隊を置いても全機が一斉攻撃しない理由

入口の実働機数に加え、目標艦隊の現在総Strengthを基準とする external-plane cap がある。一時海戦では原則として `max(20, 総Strength×2.5%)` が今回参加可能な上限になる。stacking式も残るが、MODの無補正枠は10000機なので通常規模では事実上働かない。

### 目標空母の戦闘機が役に立つ理由

目標艦隊そのものが一時海戦へ入るため、そこに搭載された艦上戦闘機は海戦用CAP処理から参照される。これは戦略地域の基地戦闘機による通常迎撃とは別の、艦艇到達直前の第二防空層である。

### AAの効果が一種類ではない理由

AAには少なくとも二つの役割がある。

1. 選ばれた目標艦自身のAAが、確率判定後に攻撃機を直接撃墜する。
2. 目標艦AAと艦隊全体AAが、実際に通った爆撃のダメージを最大66%軽減する。

したがって「航空機損失が増える効果」と「艦が受けるダメージを減らす効果」を分けて考える必要がある。

---

## 20. 確度と未確定部分

高い確度で確認できたもの:

- MODの6時間defineを2倍して12時間周期にする処理。
- 地域IDで処理位相をずらす処理。
- Naval Strike、Kamikaze、Port Strikeの任務分岐。
- 艦隊可視性、0.5のbalance factor、地域航空探知効率を使う発見抽選。
- 海戦外艦隊に一時 `CCombatNaval` を作り、解決後すぐ破棄する処理。
- external-plane cap と、MODでは日数増加係数が0であること。
- `naval_fire_exchange_air.cpp` の目標抽選、CAP、AA、命中、ダメージ式を再利用すること。
- 任務ビットをNaval Attack／Targeting getterへ渡すこと、表示値を0.1倍して内部値へ直すこと。
- 発進元空母が同じcombatの参加艦として見つかった場合だけ空母倍率を掛けること。
- 各主要段階の同期乱数呼び出し。

名前・意味に一部推定を含むもの:

- 逆コンパイル上でシンボルが失われた一部modifier IDの正式なスクリプト名。
- `FUN_140f3c8b0` 内の個々の固定小数フィールドについて、UI上のどの効率表示へ一対一対応するか。
- 候補が複数艦隊にまたがる場合の、全候補間の細かな消費順・優先順位。

ただし、これらは「12時間周期で探索し、発見後は一時海戦を作って共通strike式で解決する」という中心結論には影響しない。
