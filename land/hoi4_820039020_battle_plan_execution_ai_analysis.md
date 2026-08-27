# Innovative Balance：戦闘計画の攻勢実行AI解析

調査対象：Workshop ID `820039020`、MODバージョン `13.00`、対応表記 `1.17.5.2`  
実行ファイル：ローカルの `hoi4.exe` をGhidraで静的解析  
調査日：2026-08-26

## 結論

攻勢実行ボタンは、割り当てた全師団へ一斉に「正面の敵を攻撃せよ」と命令するだけのswitchではない。ボタンはOrderInstanceを実行状態へ移し、その後のunit controller更新で、各師団について個別に次を決める。

```text
1. その師団が現在攻撃可能か
2. 既存の隣接戦闘へ参加・支援すべきか
3. 新規攻撃候補となる隣接provinceはどれか
4. execution modeの閾値を満たす候補はどれか
5. 候補ごとの戦闘balance・進撃方向・混雑度をscore化
6. 最もscoreが高いprovinceへattack orderを発行
7. 有効な候補がなければ待機またはfront内で再配置
```

したがって、攻勢ボタンを押していても全師団が攻撃するとは限らない。ORG、Strength、supply、fort、予想戦闘balance、現在位置、front assignment、既存戦闘の進捗、attack先の混雑、orderの経路などを通過した師団だけが攻撃する。

execution modeの内部名は`Careful`、`Balanced`、`Rush`であり、日本語UIの慎重・均衡・積極的に相当する。modeは主として「攻撃を許可する条件」を変える。一方、offensive lineとspearhead（内部名`blitz_line`）は主として「どのprovinceを進撃経路として優先するか」を変える。両者は別軸である。

## 1. 調査対象の区別

### 1.1 プレイヤーが押す攻勢実行と国家AI

このレポートで扱うのは、プレイヤーまたはAIが作成したfront lineとoffensive orderを実行状態にした後、unit controllerが師団へ自動命令を出す処理である。

国家AIが「そもそも今このplanを実行開始すべきか」を決める戦略判断には、`PLAN_VALUE_TO_EXECUTE`や`PLAN_ACTIVATION_SUPERIORITY_AGGRO`など別のdefinesがある。プレイヤーが実行ボタンを押した後の各師団・各provinceの選択とは区別する必要がある。

### 1.2 用語

- front line：師団を配置する現在の正面。
- offensive line：到達させたい広い終端線。内部表示名は`offence_line`。
- spearhead：狭い経路を優先する先鋒線。内部名は`blitz_line`。
- execution mode：Careful、Balanced、Rushの攻撃許可設定。
- active attack：師団に実際の攻撃移動commandが発行された状態。

## 2. 全体の呼び出し構造

主要な流れは次のとおりである。

```text
OrderInstanceが実行中
  ↓
FUN_141457c60
  front provinceのimportanceを計算し、師団をfront segmentへ割り当てる
  ↓
FUN_1414536e0
  orderへ属する師団を走査し、現在の行動を更新する
  ↓
FUN_141455620
  既存の隣接戦闘へ参加・支援できるかを先に検討する
  ↓ 既存戦闘へ入らない場合
FUN_141448920
  新規attack先となる隣接provinceを列挙・filter・score化する
  ↓
FUN_1414452f0
  進撃方向、隣接関係、他師団の割当、fortなどから最終scoreを作る
  ↓
FUN_1414444f0 / FUN_141444390
  選ばれたprovinceへの自動commandを構築する
```

この処理は「軍全体で攻撃provinceを一個選び、全師団をそこへ送る」方式でも、「各師団がmap全体から自由に目的地を選ぶ」方式でもない。各師団は、自身の現在位置または割り当てfront segmentから到達可能な局所候補を評価する。

## 3. 攻撃する師団の決定

### 3.1 mode別ORG・Strength下限

OrderInstanceの`+0x180`がexecution modeである。UI側とGhidra上の分岐は次のように対応する。

| 内部値 | 内部名 | 日本語上の意味 | 最低ORG | 最低Strength |
|---:|---|---|---:|---:|
| 0 | `Careful` | 慎重 | 85% | 60% |
| 1 | `Balanced` | 均衡 | 70% | 50% |
| 2 | `Rush` | 積極的 | 45% | 30% |

MOD側のdefinesは次の値である。

```lua
PLAN_ATTACK_MIN_ORG_FACTOR_LOW = 0.85
PLAN_ATTACK_MIN_STRENGTH_FACTOR_LOW = 0.60

PLAN_ATTACK_MIN_ORG_FACTOR_MED = 0.70
PLAN_ATTACK_MIN_STRENGTH_FACTOR_MED = 0.50

PLAN_ATTACK_MIN_ORG_FACTOR_HIGH = 0.45
PLAN_ATTACK_MIN_STRENGTH_FACTOR_HIGH = 0.30
```

`FUN_140c030c0`はmodeからORG下限を返し、`FUN_140c03140`はStrength下限を返す。`FUN_1414598a0`と`FUN_141455620`は、師団の現在ORG ratioとStrength ratioをこれらに比較する。

ここで重要なのは、積極的modeでもORG 0%近くまで無条件で突撃するわけではない点である。このMODでは45% ORG、30% Strengthが通常の自動攻撃下限として残る。

### 3.2 supply gate

このMODのmode別supply checkは次のとおりである。

```lua
-- order: careful, balanced, rush, skip, rush_weak
PLAN_EXECUTE_SUPPLY_CHECK = { 1.0, 0.0, 0.0, 1.0, 0.0 }
```

`FUN_1414598a0`はmodeをindexとしてこの配列を読み、`FUN_140c135a0`へ渡す。

- Careful：必要supplyに対して実質100%の充足を要求する。
- Balanced：この専用gateではsupply下限を要求しない。
- Rush：この専用gateではsupply下限を要求しない。

BalancedとRushでsupplyの影響が完全に消えるという意味ではない。実際の戦闘能力、移動、attrition、戦闘balance評価にはsupply不足の結果が反映される。ただし、Carefulだけが「supply不足そのものを理由にattack commandを止める」追加gateを持つ。

### 3.3 Carefulのfort gate

```lua
PLAN_EXECUTE_CAREFUL_MAX_FORT = 5
```

`FUN_141448920`と`FUN_141445d20`はtarget provinceのfort levelを読み、Carefulの場合、defenderが存在するlevel 5以上のfortへの自動攻撃を拒否する。

判定は概念的に次の形である。

```text
if mode == Careful
and target_fort_level >= 5
and target_has_defending_units:
    do_not_auto_attack
```

無人provinceの占領まで禁止する条件ではない。またBalancedとRushには、この専用fort上限は適用されない。fortは別途target scoreや予想戦闘balanceを悪化させるため、上限がなくても攻撃しにくくはなる。

### 3.4 その他の共通条件

ORG・Strengthだけを満たしても攻撃は確定しない。Ghidra上では少なくとも次も確認される。

- 師団が当該orderへ割り当てられていること。
- frontまたは進撃経路との対応が取れていること。
- retreat、移動不能、無効なunit stateなど、attack commandを作れない状態でないこと。
- targetが隣接または有効なpathで到達可能であること。
- 外交・controller関係上、敵対provinceとして攻撃可能であること。
- 師団がfront上の担当segmentから大きく外れている場合、先に再配置すること。
- 有効なattack候補が一個以上残ること。

つまり「積極的にすれば全部隊が常時攻撃」ではなく、modeで緩和されるのは複数gateの一部である。

## 4. 新規攻撃より先に既存戦闘への参加を検討する

`FUN_1414536e0`は、直ちに新しいtargetを探す前に`FUN_141455620`を呼ぶ。この関数は、隣接provinceで既に進行している戦闘へ当該師団を参加・支援させるべきかを検討する。

```lua
PLAN_MAX_PROGRESS_TO_JOIN = 0.50
```

Ghidraでは戦闘progress値と`0.50`を比較し、進捗がこの値以上の戦闘は追加支援の優先対象から外す分岐がある。defineのcommentどおり、progressが低く、まだ支援を必要とする戦闘を優先するための値である。

既存戦闘への参加でも、同じmode別ORG・Strength下限が使用される。さらに、両軍の参加師団数・戦力差、当該師団が戦闘へ到達できるかなどを確認する。

したがって自動攻勢は、毎回新しいprovinceへ戦線を拡大するのではなく、まず既存の苦戦中の戦闘へ近隣師団を追加することがある。ただし進捗が50%以上まで進んだ戦闘へ無制限に後続師団を投入する設計ではない。

## 5. 新規attack target provinceの候補生成

既存戦闘へ参加しなかった師団について、`FUN_141448920`がtarget候補を作る。この関数は、師団の局所的な隣接province列を走査する。

概念的なfilter順は次のとおりである。

```text
for province in adjacent_or_front_connected_provinces:
    if provinceが無効・対象外:
        continue
    if orderの進撃領域・front接続と合わない:
        continue
    if 外交関係上attack不能:
        continue
    if pathまたは地形上到達不能:
        continue
    if Careful fort gateに違反:
        continue

    route_score = order上の進捗・方向score

    if route_score <= execution_mode_limit:
        continue

    combat_balance = estimate_combat_balance(...)
    if non-Rush and combat_balance <= 0.20:
        continue

    final_score = score_candidate(...)
    最も高い候補を保存
```

候補はmap全体から探すのではないため、同じ軍に所属する全師団が同じtarget集合を見るわけではない。現在位置、担当front segment、order pathによって候補集合が異なる。

## 6. execution modeがprovince選択へ与える差

このMODの候補score下限は次のとおりである。

```lua
PLAN_EXECUTE_CAREFUL_LIMIT = 25
PLAN_EXECUTE_BALANCED_LIMIT = 0
PLAN_EXECUTE_RUSH = -200
MIN_BALANCE_SCORE_TO_PROCEED_ATTACK = 0.2
```

コードの比較は概念的に`candidate_route_score > limit`であり、同値では通らない。

| mode | route score条件 | 予想combat balance下限 | supply専用gate | fort専用gate |
|---|---:|---:|---:|---:|
| Careful | `> 25` | `> 0.20` | 100% | level 5以上を拒否 |
| Balanced | `> 0` | `> 0.20` | なし | なし |
| Rush | `> -200` | 専用下限をbypass | なし | なし |

このため積極的modeは、単にORG・Strength下限が低いだけではない。

1. order上で価値が低い、あるいは多少不利な方向のprovinceも候補に残す。
2. `MIN_BALANCE_SCORE_TO_PROCEED_ATTACK`による0.20の足切りをbypassする。
3. supplyとfortの慎重専用gateを使用しない。

一方で、外交・到達可能性・order接続・ORG 45%・Strength 30%などの共通条件は残る。

## 7. candidate provinceの最終score

mode閾値を通過した候補について、`FUN_1414452f0`を中心に最終scoreを作る。完全な変数名は実行ファイルから失われているが、次の要素はコード上で確認できる。

### 7.1 order上の進撃方向

provinceがoffensive orderのinterpolated frontや進撃pathにどれだけ沿っているかを基礎scoreとして使う。描画した攻撃線と無関係な横方向へ常に進むわけではない。

### 7.2 複数方向から攻撃できる位置

targetへ接続する味方側province、同じfront/orderに属する隣接関係などが加点される。複数方向から到達可能なprovinceは、孤立した一方向の候補より高くなり得る。

### 7.3 fort

`FUN_1414452f0`のreturn直前では、構築したscoreを概ね`fort_level + 1`で割る。Carefulのhard gateとは別に、全modeで高fortがtarget desirabilityを下げる。

```text
final_geometry_score
≈ accumulated_score / (fort_level + 1)
```

### 7.4 予想戦闘balance

`FUN_140e42740`がtargetに対する戦闘balanceを評価し、`FUN_140e42600`が動的なattack modifierを集計する。師団のSoft Attackを単独で比較する単純な処理ではなく、既存参加部隊、敵、combat modifierを含む評価値がtarget選択へ入る。

### 7.5 attackの分散

```lua
PLAN_SPREAD_ATTACK_WEIGHT = 12.0
```

同じ更新cycle内ですでに他師団が同じtarget provinceを選んでいる場合、その個数を数え、候補scoreを大きく下げる。概念的には次の傾向を持つ。

```text
adjusted_score
≈ raw_score / (existing_assignments_to_target × PLAN_SPREAD_ATTACK_WEIGHT)
```

厳密には固定小数点変換、最低値、blitz contextの係数が追加される。このpenaltyにより、通常のoffensive lineでは全師団が一個のprovinceへ集中しにくく、複数の隣接provinceへattackを分散しやすい。

### 7.6 最大scoreを選ぶ

`FUN_141448920`はbest scoreを`-10000`で初期化し、候補を順に比較して、より大きいものだけを更新する。したがって基本構造はweighted randomではなくmaximum selectionである。同scoreの場合は候補の走査順が結果へ影響し得る。

## 8. 小規模な無抵抗pocket

```lua
PLAN_MIN_AUTOMATED_EMPTY_POCKET_SIZE = 2
```

敵unitが存在せず、規模が二province以下の小pocketについては、候補scoreを`255`へ引き上げる特別分岐がある。これにより、通常の進撃方向から多少外れても、小さな無抵抗pocketを自動処理しやすい。

これは敵が守る大きな包囲地域を常に自動掃討する規則ではない。無抵抗で小さいことが条件である。

## 9. offensive lineとspearheadの違い

### 9.1 共通部分

どちらもfront lineからdestinationへ向かうoffensive orderであり、次は共通する。

- 師団ごとのORG・Strength gate
- execution modeのscore閾値
- supply・fort条件
- 隣接・到達可能性の確認
- 既存戦闘への参加判定
- 最終的なattack commandの発行

spearheadを使っても、師団のcombat statが直接増えるわけではない。また戦車だけを自動選別するhard-coded条件も、今回追跡したattack decisionにはない。どの師団が使われるかはorderへ割り当てた師団とfront assignmentによる。

### 9.2 offensive line

通常のoffensive lineは、広いdestination lineへ向けて中間frontを補間し、前進に伴ってfrontを再構築する。

```lua
FRONTLINE_EXPANSION_FACTOR = 0.6
```

`FUN_14100c4e0`はinterpolated frontの座標列を再構築するとき、この値を使用して進撃先の端を外方向へ拡張する。define commentのとおり、0なら描画した終端へ比較的直進し、値が大きいほど前進中に横のprovinceを取り込んでfrontを広げる。

このMODの0.6では、offensive lineは終端線への最短経路だけを通るのではなく、連続した広いfrontを保つ方向へかなり拡張する。加えて`PLAN_SPREAD_ATTACK_WEIGHT = 12`が同じtargetへの過集中を抑えるため、複数正面への分散攻撃になりやすい。

### 9.3 spearhead / blitz line

Ghidraとserialization文字列ではspearheadは`blitz_line`として扱われる。OrderInstanceには通常のfront interpolationとは別に、`_BlitzProvinces`という専用province listが存在し、`+0x320`のvectorへ保存される。

target selectorはcandidate provinceがこのlistに含まれる場合、route scoreを`300`へ設定する。これはCareful 25、Balanced 0、Rush -200の全閾値を大きく上回る。

```text
if candidate in order._BlitzProvinces:
    route_score = 300
```

さらにblitz pathが有効な場合、予想戦闘balanceへ次を加える。

```lua
PLAN_BLITZ_OPTIMISM = 0.2
```

```text
evaluated_combat_balance
= 0.20
```

これは実戦時のAttack・Breakthroughへ直接+20%するcombat bonusではない。自動計画AIが「このtargetなら進んでよい」と判断する際だけ、攻撃側に有利なものとして0.20楽観的に評価する補正である。

blitz contextでは、同じtargetへ既に割り当てられたattack数をspread penaltyへ渡す前に二倍する分岐もある。そのため専用経路で候補を狭めつつ、一個のprovinceだけに全師団を過剰投入することは別途抑制する。

### 9.4 実質的な違い

| 項目 | offensive line | spearhead / blitz line |
|---|---|---|
| 進撃形状 | 広いfrontを補間・拡張 | 専用`_BlitzProvinces`を強く優先 |
| 横方向への拡張 | `FRONTLINE_EXPANSION_FACTOR`の影響が大きい | 専用経路による拘束が強い |
| 経路provinceのroute score | 通常のorder progressから算出 | list内なら300 |
| 戦闘balance評価 | 通常値 | `PLAN_BLITZ_OPTIMISM`で+0.20 |
| 同一targetへの混雑penalty | 通常 | countを二倍する分岐あり |
| combat statへの直接bonus | なし | なし |

短く言えば、offensive lineは「線全体を押し上げる」orderで、spearheadは「指定した狭いprovince列をAI評価上の最優先経路として押し込む」orderである。

## 10. execution modeとorder typeは別軸

よくある誤解は、spearheadを選ぶだけで常に積極的modeになるというものだが、コード上は別field・別分岐である。

例としてspearhead + Carefulなら、次が同時に成立する。

- `_BlitzProvinces`はroute score 300で強く優先される。
- combat balance評価へ+0.20される。
- ただし師団はORG 85%以上、Strength 60%以上を要求される。
- supply専用gateは100%を要求する。
- defenderのいるlevel 5以上のfortは拒否する。
- 非Rushなので、補正後combat balanceが0.20を超える必要がある。

逆にoffensive line + Rushなら広いfrontへ分散する形のまま、route score -200、ORG 45%、Strength 30%まで条件を緩め、balance 0.20の足切りをbypassする。

したがって、進撃経路を決めるのがorder type、損害をどこまで許容してattack commandを出すかを決めるのがexecution mode、と理解するとよい。

## 11. 具体例

師団Aがfront provinceから敵province X・Yへ隣接していると仮定する。

### 11.1 Careful

- 師団A：ORG 80%、Strength 90%
- X：route score 50、balance 0.5、fort 0
- Y：route score 100、balance 0.8、fort 0

どちらのprovinceが良くても、師団AはORG 85%未満なので攻撃しない。

### 11.2 Balanced

同じ師団AはORG 70%以上、Strength 50%以上を満たす。X・Yはroute score 0を超え、balance 0.20も超えるため候補になる。最終geometry score、既存attack数、fortなどを比較し、高い方を選ぶ。

### 11.3 Rush

Xのbalanceが0.10まで悪化しても、Rushは0.20 gateをbypassするため候補に残り得る。ただしORG 45%・Strength 30%未満なら通常の自動攻撃は止まる。

### 11.4 spearhead

Yが`_BlitzProvinces`に含まれていればroute scoreは300になる。またbalance評価へ0.20が加わるため、通常offensive lineでは落とされる候補でも通過しやすい。ただしmodeがCarefulなら、ORG・Strength・supply・fort条件はそのまま残る。

## 12. 実戦上の意味

1. 積極的modeは「攻撃頻度」だけでなく、低ORG・低Strength・低route score・低balanceの攻撃を許可する複数のgate変更である。
2. Carefulは補給・fort・戦闘balanceの条件が重いため、実行中表示でも長時間攻撃しないことがある。
3. offensive lineで攻撃が散るのは偶然ではなく、front expansionと`PLAN_SPREAD_ATTACK_WEIGHT`による明示的な分散設計である。
4. spearheadはdamage bonusではなく、専用province path、route score 300、balance optimism +0.20によって突破方向を固定しやすくする。
5. 同じ軍の師団でも現在位置とfront segmentが違えば候補provinceが違うため、全師団が同じ弱点を自動集中攻撃するとは限らない。
6. 既存戦闘progressが50%未満なら、近隣師団は新規attackより支援参加を優先し得る。
7. target scoreはmaximum selectionだが、同点では内部の候補走査順が影響するため、完全再現にはprovince adjacency listとupdate順も必要になる。

## 13. 主要関数・field・define対応

| 役割 | アドレス / field |
|---|---|
| order/front全体のimportance・配置更新 | `FUN_141457c60` @ `0x141457c60` |
| order所属師団の行動更新 | `FUN_1414536e0` @ `0x1414536e0` |
| 既存戦闘への参加・支援判断 | `FUN_141455620` @ `0x141455620` |
| 新規target provinceの列挙・filter・最大score選択 | `FUN_141448920` @ `0x141448920` |
| targetのgeometry・混雑・fort score | `FUN_1414452f0` @ `0x1414452f0` |
| combat balance評価 | `FUN_140e42740` @ `0x140e42740` |
| dynamic attack modifier集計 | `FUN_140e42600` @ `0x140e42600` |
| 師団の実行可能性・supply・ORG・Strength gate | `FUN_1414598a0` @ `0x1414598a0` |
| mode別ORG下限 | `FUN_140c030c0` @ `0x140c030c0` |
| mode別Strength下限 | `FUN_140c03140` @ `0x140c03140` |
| mode別supply check | `FUN_140c135a0` @ `0x140c135a0` |
| front interpolation・expansion | `FUN_14100c4e0` @ `0x14100c4e0` |
| OrderInstance serialization、`_BlitzProvinces` | `FUN_14100aec0` @ `0x14100aec0` |
| execution mode | OrderInstance `+0x180` |
| blitz line種別flag | OrderInstance `+0x118` |
| runtime blitz connection判定 | connected order field `+0x248` |
| `_BlitzProvinces` vector | OrderInstance `+0x320`、count `+0x32c` |
| `PLAN_EXECUTE_CAREFUL_LIMIT` | `DAT_14324b770` |
| `PLAN_EXECUTE_BALANCED_LIMIT` | `DAT_14324b7f4` |
| `PLAN_EXECUTE_RUSH` | `DAT_14324b8a0` |
| `PLAN_EXECUTE_SUPPLY_CHECK` | `DAT_14324d0e8` |
| `PLAN_EXECUTE_CAREFUL_MAX_FORT` | `DAT_14324b924` |
| `PLAN_MAX_PROGRESS_TO_JOIN` | `DAT_14324bb98` |
| `MIN_BALANCE_SCORE_TO_PROCEED_ATTACK` | `DAT_1432476b0` |
| `PLAN_SPREAD_ATTACK_WEIGHT` | `DAT_14324a1b0` |
| `PLAN_BLITZ_OPTIMISM` | `DAT_14324bc30` |
| `FRONTLINE_EXPANSION_FACTOR` | `DAT_143246f80` |

## 14. 確度と未確定部分

### GhidraとMOD definesから直接確認済み

- execution modeの内部値0/1/2とCareful/Balanced/Rushの対応
- mode別ORG・Strength下限
- Carefulだけが使用するsupply 1.0 gate
- Carefulのfort level 5 gate
- province route score閾値25/0/-200
- 非Rushのcombat balance 0.20 gateとRushのbypass
- 既存戦闘progress 0.50による追加参加判断
- target候補を師団ごとの局所provinceから列挙すること
- 最大scoreのcandidateを選ぶこと
- 同じtargetへの割当数と`PLAN_SPREAD_ATTACK_WEIGHT`によるscore低下
- `_BlitzProvinces`の保存fieldと、list内candidateのscore 300
- blitz contextの`PLAN_BLITZ_OPTIMISM = 0.2`
- front再構築時の`FRONTLINE_EXPANSION_FACTOR = 0.6`

### 型名が失われているため意味を計算形から特定したもの

- `FUN_1414452f0`内の細かな各加点が、front方向・隣接方向・他師団割当のどれに対応するかの完全な変数名
- 同score時に候補listへ入る厳密なprovince順序
- unit controller更新の正確なwall-clock間隔。これは陸戦damageの一時間tickとは別系統であるため、本稿では「controller更新ごと」とする。

このため、再現実装では大きなgateとscore構造までは確定できるが、save gameと一手単位で同じtargetを選ばせるには、front graphの生成順、province adjacency list、師団走査順まで合わせる必要がある。
