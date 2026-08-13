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
Air Defence（航空機防御）に加算  
Agility（機動性）には加算されない**

という処理になっていました。

したがって、今回解析したHOI4バージョンでは、

> **`air_power_projection_factor` は、home外で活動する航空機のAir AttackとAir Defenceを強化するmodifierであり、Agilityには影響しない。**

というのが結論です。

なお、「home」の判定に使われている場所IDが厳密にState・Strategic Regionなどのどれなのか、またPower Projectionが別の処理でも利用されているかについては、今回の解析では未確定です。

今回の調査では、単なるWikiやコミュニティの推測ではなく、`hoi4.exe` 内の **文字列 → modifier登録 → 内部index → 実際の航空能力計算**まで追跡して確認しています。