## HOI4「air_power_projection_factor」解析まとめ

HOI4には `air_power_projection_factor` というmodifierがありますが、Wikiなどの公開情報だけでは「具体的に何を強化し、何をもってhome / awayを判定するのか」がはっきりしていませんでした。そこで `hoi4.exe` をGhidraで静的解析し、modifier登録から実際の航空能力計算、地理オブジェクトの所属関係まで追跡しました。

## 確認された内部対応

modifier登録処理と航空stat取得処理から、解析対象バージョンでは次の対応を確認しました。

| 内部index | Modifier / stat | 対応 |
|---:|---|---|
| `0x6f` (111) | `MODIFIER_AIR_ATTACK_FACTOR` | Air Attack |
| `0x70` (112) | `MODIFIER_AIR_DEFENCE_FACTOR` | Air Defence |
| `0x71` (113) | `MODIFIER_AIR_AGILITY_FACTOR` | Agility |
| `0x74` (116) | `air_home_defence_factor` | 同一Strategic Region側 |
| `0x75` (117) | `air_power_projection_factor` | 別Strategic Region側 |

航空stat側では、`stat 0x30 = Air Attack`、`stat 0x2f = Air Defence`、`stat 0x31 = Agility`という対応です。

## home判定の正体

最終的に地理オブジェクトの関連付けまで追った結果、次の構造がコードから確認できました。

```text
CProvince + 0xC0 = CState*
CProvince + 0xC8 = CStrategicRegion*
CStrategicRegion + 0x58 = 比較に使われる識別値
```

`CProvince + 0xC0 = CState*` は、`state.cpp` 由来の `FUN_1409c32d0` にある重複所属警告と `province->field_C0 = thisState` 相当の代入から確認しました。

`CProvince + 0xC8 = CStrategicRegion*` は、`strategicregion.cpp` 由来の `FUN_140deb2a0` にある次の処理から直接確認しました。

- 既存の `province + 0xC8` と現在処理中のStrategic Regionを比較する
- `"Province has added another strategic region as theirs..."` という警告を出す
- 旧・新Strategic Regionそれぞれの `+0x58` を識別値として表示する
- 最後に `province->field_C8 = thisStrategicRegion` 相当の代入を行う

`+0x58` の正式なC++メンバ名はPDBなしでは未確認ですが、Strategic Regionを区別する値として警告表示と航空計算の双方で使われるため、実質的なStrategic Region IDとみなせる識別値です。

実際の航空能力計算関数 `FUN_140f23c00` は、任務対象Provinceと基地・空母側Provinceについて、各 `CStrategicRegion + 0x58` を比較します。意味的には次の処理です。

```cpp
targetProvince = GetProvince(targetProvinceId);
baseProvince   = GetProvinceOfAirBaseOrCarrier(...);

targetRegionId = targetProvince->strategicRegion->field_58;
baseRegionId   = baseProvince->strategicRegion->field_58;

if (targetRegionId != 0 && baseRegionId != 0) {
    if (targetRegionId == baseRegionId) {
        bonus = air_home_defence_factor;
    } else {
        bonus = air_power_projection_factor;
    }

    airAttackFactor  += bonus;
    airDefenceFactor += bonus;
    // agilityFactorには加算されない
}
```

したがって、ここでいうhome / awayはStateや国境ではなく、**航空隊の基地・空母と任務対象が同じStrategic Regionに属するか**で決まります。

## 補正の適用先

Home DefenceとPower Projectionは別々の計算式ではありません。Strategic Regionの比較結果に応じてどちらか一方を選び、選ばれた値を同じ経路で次の2つへ加算します。

- Air Attack factor
- Air Defence factor

Agility factorには加算されません。

内部では `100000` を基準にした固定小数点形式が使われており、概略は次の形です。

```text
Final Air Attack
= Base Air Attack × Attack Factor / 100000

Final Air Defence
= Base Air Defence × Defence Factor / 100000

Final Agility
= Base Agility × Agility Factor / 100000
```

選択されたHome DefenceまたはPower Projectionの値は、最終能力値へ独立して後掛けされるのではなく、Air Attack / Air Defenceのfactor蓄積値へ加算され、その合計factorが基礎statへ適用されます。

## 証拠レベルと留保

コード上で確認済みなのは、内部indexの対応、Provinceの `CState*` / `CStrategicRegion*` フィールド、Strategic Region識別値の比較条件、Home Defence / Power Projectionの選択、およびAir Attack / Air Defenceへの加算とAgilityへの非加算です。

未確認なのは、`CStrategicRegion + 0x58` の正式なソース上のメンバ名と、このmodifierが航空能力計算以外の別システムでも追加利用されるかどうかです。したがって「航空優勢ポイントなどに別途まったく作用しない」とまでは、この調査だけでは断定しません。

## 結論

> **`air_home_defence_factor` は航空隊の基地・空母と任務対象が同一Strategic Regionにある場合、`air_power_projection_factor` は両者が別Strategic Regionにある場合に選択される。どちらもAir AttackとAir Defenceのfactorへ同じ経路で加算され、Agilityには加算されない。**

この結論は、`hoi4.exe` 内の **文字列 → modifier登録 → 内部index → 航空能力計算 → ProvinceのState / Strategic Region所属代入 → Strategic Region識別値の比較**まで追跡した結果です。
