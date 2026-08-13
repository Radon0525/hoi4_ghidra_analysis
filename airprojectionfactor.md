## HOI4「air_power_projection_factor」解析まとめ

HOI4には `air_power_projection_factor` というmodifierがありますが、Wikiなどの公開情報だけでは「具体的に何を強化しているのか」がはっきりしていませんでした。そこで `hoi4.exe` をGhidraで静的解析し、実際のゲーム内部処理を追跡しました。

まず文字列 `air_power_projection_factor` からXREFを辿り、modifier登録処理を解析した結果、内部では以下の対応になっていることを確認しました。

- `0x74` = `air_home_defence_factor`
- `0x75` = `air_power_projection_factor`
- `0x6f` = `AIR_ATTACK_FACTOR`
- `0x70` = `AIR_DEFENCE_FACTOR`
- `0x71` = `AIR_AGILITY_FACTOR`

その後、実際の航空隊能力を計算している関数 `FUN_140f23c00` に到達しました。

ここでは、ある2つの「場所ID」を比較し、

```text
同じ場所 → air_home_defence_factor
違う場所 → air_power_projection_factor
```

という分岐が行われています。つまりPower Projectionは、いわゆる「homeではない場所」で活動するときの補正であることがコード上確認できました。

さらに重要なのが、その補正の適用先です。

Power Projectionは、

**Air Attack（対空攻撃）に加算  
Air Defence（航空機防御）に加算**

という処理になっていました。

### 数式として見ると

内部ではおおむね、

```text
最終 Air Attack
= 基礎 Air Attack × Attack Factor

最終 Air Defence
= 基礎 Air Defence × Defence Factor


という構造で能力値を計算しています。

実際のコードでは `100000` を基準にした固定小数点形式が使われており、概略としては、

```text
Final Attack
= Base Attack × AttackFactor / 100000

Final Defence
= Base Defence × DefenceFactor / 100000

となっています。

そしてaway判定の場合、`air_power_projection_factor` の値が **Attack FactorとDefence Factorの両方へ加算**されます。

つまり概念的には、

```text
Attack Factor
= 通常のAttack補正 + その他の補正 + Power Projection

Defence Factor
= 通常のDefence補正 + その他の補正 + Power Projection

です。

したがってPower Projectionは、最終能力値に最後から独立して `×1.10` するような別枠乗算ではなく、**既存のAir Attack / Air Defenceのfactor群に加算され、その合計factorが基礎能力値へ掛けられる**仕組みと読めます。

### 結論

今回解析したHOI4バージョンでは、

> **`air_power_projection_factor` は、homeではない場所で活動する航空機のAir AttackとAir Defenceを強化するmodifierである。**

また、`air_home_defence_factor` も同じAir Attack / Air Defenceに作用し、場所判定によってHome DefenceとPower Projectionのどちらを使うか切り替えていることが確認できました。

なお、「home」の判定に使われている場所IDが厳密にState・Strategic Regionなどのどれなのか、またPower Projectionが別の処理でも利用されているかについては、今回の解析では未確定です。

今回の調査では、Wikiやコミュニティの推測ではなく、`hoi4.exe` 内の **文字列 → modifier登録 → 内部index → 実際の航空能力計算 → Attack / Defence / Agilityへの適用**まで追跡して確認しています。
