# HOI4 海軍任務解析資料

> 対象: HOI4 1.17系の提供Ghidraプロジェクトと、Workshop `820039020`（innovative balance v13.00）

海軍任務を継続して解析するための専用フォルダ。各資料は、対象exeの実行経路とMODの `common/defines/00_defines.lua` を照合している。

## 資料一覧

| 任務 | 資料 | 中心内容 |
|---|---|---|
| 哨戒 | [hoi4_naval_patrol_mission_system.md](./hoi4_naval_patrol_mission_system.md) | 通常艦隊の索敵、純潜水艦の初回抽選、0～100%発見進捗、航空・レーダー寄与 |
| 打撃部隊 | [hoi4_naval_strike_force_mission_system.md](./hoi4_naval_strike_force_mission_system.md) | 完全発見／既存海戦からの目標割当、交戦規定、港からの出撃、海戦参加 |
| 船団護衛 | [hoi4_naval_convoy_escort_mission_system.md](./hoi4_naval_convoy_escort_mission_system.md) | 船団カバー率、海域カバー率、1.7乗、輸送船海戦への増援 |
| 船団襲撃 | [hoi4_naval_convoy_raiding_mission_system.md](./hoi4_naval_convoy_raiding_mission_system.md) | 任務効率、通常／部隊／上陸輸送の多段発見抽選、追跡進捗、海戦生成 |
| 輸送船・輸送航路 | [hoi4_convoy_transport_and_route_efficiency_system.md](./hoi4_convoy_transport_and_route_efficiency_system.md) | 撃沈後の海域効率低下、必要／配属船団、資源・補給への波及、完全遮断と回復式 |

## 任務番号

対象exeの `FUN_140f7a2c0` にある名称テーブルと `FUN_140f80a70` の分岐を照合した対応。

```text
0 HOLD
1 PATROL
2 STRIKE_FORCE
3 CONVOY_RAIDING
4 CONVOY_ESCORT
5 MINES_PLANTING
6 MINES_SWEEPING
7 TRAINING
8 RESERVE_FLEET
9 NAVAL_INVASION_SUPPORT
```

旧版の哨戒資料にあった `0 = Strike Force` は誤りで、修正済みである。

## 4任務の役割分担

```text
哨戒
  敵艦隊を追跡し、観測国の共有発見状態を0→3へ進める

打撃部隊
  完全発見された敵または既存海戦を割り当てられ、移動後に海戦へ参加する

船団襲撃
  輸送船を独自の多段抽選と0～100%進捗で捕捉し、新しい輸送船海戦を作る

船団護衛
  抽象的な護衛効率を作り、味方輸送船海戦が発生した後に増援として参加する
```

このため、哨戒と打撃部隊は一組になりやすい。一方、船団襲撃と船団護衛は、通常艦隊用の共有発見状態だけではなく、輸送航路と輸送船海戦を中心とする専用処理を持つ。

## MODで特に重要な式

### 船団襲撃効率

coordinationと追加modifierを省略すると、MODでは次になる。

```text
Eraid = clamp(襲撃任務部隊数 / 担当海域数, 0, 1)^1.7
```

1任務部隊を2海域へ出すと約30.8%、3海域なら約15.5%。艦数を増やしても、この海域カバー不足は直接解消しない。

### 船団護衛効率

coordinationと追加modifierを省略すると次になる。

```text
Econvoy = clamp(護衛艦数 × 25 / 保護対象輸送船数, 0, 1)
Eregion = clamp(護衛任務部隊数 × 5 / 担当海域数, 0, 1)

Eescort = (Econvoy × Eregion)^1.7
```

MODはバニラの12輸送船／隻を25へ強化しているが、5海域／任務部隊は同じである。

## 資料を読む際の注意

- `AGGRESSION_LEVEL_BY_MISSION_*` や `*_TARGET_RECALC_DAYS` は主にAIの編成・任務配置用で、プレイヤー任務の毎時ハード条件とは区別する。
- `NAVAL_STRIKE_FORCE_ATTACK_LIKELYHOOD_THR_*` はUI表示閾値であり、打撃部隊の出撃式そのものではない。
- defineコメントだけでなく、値が実際に使われるexe関数まで追えたものを高確度としている。
- 固定小数点は多くの海軍処理で `100000 = 1.0`。発見進捗の100%閾値は内部値 `10000000`。
