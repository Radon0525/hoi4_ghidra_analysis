かなり長かったので、「環境構築 → Ghidra解析 → modifier登録機構の解明 → 実際の航空計算へ到達 → Air Attack / Defence / Agility の特定」まで、証拠の強さも分けながら全部まとめます。
あとで見返しても解析を再開できるように、関数名・アドレス・内部IDも残します。
0. 今回の最終結論
最初に今回いちばん重要なところだけ書くと、
air_power_projection_factor の正体
HOI4内部では、
「航空隊の基地・空母と任務対象が別のStrategic Regionにある場合に、Air Attack と Air Defence のfactorへ加算されるmodifier」
として処理されています。
今回の解析で確認できた対応は、
内部index	Modifier	対応機体stat	Power Projectionの影響
0x6f = 111	MODIFIER_AIR_ATTACK_FACTOR	Air Attack	あり
0x70 = 112	MODIFIER_AIR_DEFENCE_FACTOR	Air Defence	あり
0x71 = 113	MODIFIER_AIR_AGILITY_FACTOR	Agility	なし
0x74 = 116	air_home_defence_factor	特殊補正	同一Strategic Region側
0x75 = 117	air_power_projection_factor	特殊補正	別Strategic Region側


実際の処理は意味的にはほぼ、
targetRegionId = targetProvince->strategicRegion->field_58;
baseRegionId   = baseProvince->strategicRegion->field_58;

if (targetRegionId != 0 && baseRegionId != 0) {
    if (targetRegionId == baseRegionId) {
        bonus = air_home_defence_factor;
    }
    else {
        bonus = air_power_projection_factor;
    }

    air_attack_factor  += bonus;
    air_defence_factor += bonus;

    // agility_factor には加えない
}
です。
この判定に使われる地理構造も、関連付け処理から次のように確認できました。
CProvince + 0xC0 = CState*
CProvince + 0xC8 = CStrategicRegion*
CStrategicRegion + 0x58 = 比較対象となる識別値
`+0x58` の正式なC++メンバ名はPDBなしでは未確認ですが、Strategic Regionを区別する値として警告表示と航空計算の両方で使われています。
したがって、少なくともこの航空能力計算においては、
Power Projection → Air Attack と Air Defenceを上昇
Power Projection → Agilityには影響しない
同一Strategic Region → Home Defence
別Strategic Region → Power Projection
ことまで突き止めました。
1. そもそものスタート地点
最初はGhidraそのものを入れるところからでした。
Javaについては、
javac 21.0.12
が動作していて、
C:\Program Files\Eclipse Adoptium\jdk-21.0.12.8-hotspot\bin\java.exe
が使われていました。
途中で、
Error: Could not create the Java Virtual Machine.
Error: A fatal exception has occurred.
も出ましたが、最終的にはJDK 21もGhidraも正常起動。
そこから hoi4.exe をGhidra ProjectへImportしました。
2. 最初の大事故：Auto Analysisが終わらない
HOI4の hoi4.exe をAuto Analyzeしたところ、
RTTI0 references Search
や
Stack FUN_...
が延々続きました。
特にStack Analyzerでは、
83% (97324 of 117470)
みたいに約11万7千関数を処理していました。
ここで分かった教訓は、
HOI4級の巨大C++バイナリをGhidraのデフォルト解析全部ONで回すと、RTTIやStack解析が猛烈に重い。

ということ。
途中キャンセルして再解析したため、PDB/Demangler系の警告も大量に出ました。
代表例：
PDB Universal> Warning: Could not find an appropriate PDB file
Demangler Microsoft> Apply failure
ただしこれは、今回の解析を不可能にする致命的エラーではありませんでした。
MFC系、
CMFCRibbonBar
CMFCToolBar
CWinApp
などのシンボルでApply failureが出ていただけで、HOI4本体のコード解析は普通に可能でした。
3. 最初の標的：air_power_projection_factor
本題です。
Ghidraの
Search → Memory
から、
air_power_projection_factor
を検索。
2件ヒットしました。
一つは大文字の、
MODIFIER_AIR_POWER_PROJECTION_FACTOR
もう一つは小文字の、
air_power_projection_factor
でした。
小文字側のアドレスは、
142a14c78
ラベルは、
s_air_power_projection_factor_142a14c78
でした。
しかもすぐ上に、
air_home_defence_factor
がありました。
これは最終的に非常に重要になります。
4. 最初のXREF
air_power_projection_factor のXREFは1件。
FUN_1400810b0 : 1400aaed7
へ飛びました。
そこで見えたのが、
MOV EDX,0x4d99
...
CALL FUN_1401467f0
...
LEA R8,[s_air_power_projection_factor_142a14c78]
MOV EDX,dword ptr [RAX]
LEA RCX,[DAT_1432d6bb0]
CALL FUN_142412a40
です。
最初は、
0x4d99 がmodifier IDなのか？

という疑問が出ました。
5. FUN_142412a40 の解析
FUN_142412a40 をDecompilerで見ると、
*param_1 = 0;
*(undefined1 *)(param_1 + 1) = 0;
FUN_140140870(param_1 + 2);

...

while (*(char *)(param_3 + lVar2) != '\0');

...

*param_1 = param_2;

...

FUN_1424d7850(lVar1,param_3,lVar2);
*(undefined1 *)(lVar2 + lVar1) = 0;
という処理。
要するに、
オブジェクト初期化
整数 param_2 を保存
param_3 の文字列長を測る
文字列をコピー
終端 \0 を付加
という処理です。
概念的には、
init_descriptor(
    destination,
    integer_id,
    name_string
);
のようなもの。
したがって、
DAT_1432d6bb0
は air_power_projection_factor 用のdescriptor的オブジェクトだと考えられました。
6. DAT_1432d6bb0 を追ったが、ここは行き止まり
DAT_1432d6bb0 のXREFは2件だけ。
1400aaee0
1425f3ad9
上は先ほどの初期化。
下へ行くと、
void FUN_1425f3ad0(void)
{
    FUN_14014bea0(&DAT_1432d6bb0);
    return;
}
だけでした。
つまり、
初期化
↓
DAT_1432d6bb0
↓
終了時に破棄
というだけ。
実際の航空計算はこのグローバルオブジェクトを直接参照していないらしい、と分かりました。
ここで方向転換。
7. 0x4d99 を追跡
先ほどの、
MOV EDX,0x4d99
CALL FUN_1401467f0
が気になったので、
FUN_1401467f0
をDecompiler。
結果は驚くほど単純でした。
undefined4 * FUN_1401467f0(undefined4 *param_1,undefined4 param_2)
{
    *param_1 = param_2;
    return param_1;
}
つまり、
ただ整数をコピーするだけ。
なので、
0x4d99
は途中で変換されているわけではありません。
そのまま air_power_projection_factor のdescriptorへ入っています。
8. Scalar Search 0x4d99
そこでGhidraのScalar Searchで、
0x4d99
を全exe検索。
かなり時間がかかりました。
結果は5件。
1400aaec5   FUN_1400810b0

14057e8b1   FUN_140543c60
14057e98e   FUN_140543c60
14057ea6b   FUN_140543c60
14057eb48   FUN_140543c60
つまり、
初期化1件＋FUN_140543c60 内4件
でした。
これによって、
FUN_140543c60
がmodifier登録システムの巨大関数らしい、と分かりました。
9. 大文字のLocalization keyも登場
FUN_140543c60 内の 0x4d99 周辺を見ると、
LEA RDX,[s_MODIFIER_AIR_POWER_PROJECTION_FACTOR...]
LEA RCX,[...]
CALL FUN_140119680
がありました。
つまり、
MODIFIER_AIR_POWER_PROJECTION_FACTOR
もここで扱われています。
そして、
MOV EDX,0x4d99
MOV RCX,[...]
CALL FUN_1401467f0
で 0x4d99 を一時オブジェクトへ格納。
その後さらに処理が続き、
MOV EDX,0x75
MOV ECX,dword ptr [RAX]
CALL FUN_14053b840
へ到達しました。
ここで初めて、
0x4d99
とは別に、
0x75
が登場。
10. FUN_14053b840 がmodifier registry登録関数だった
Decompiler全文から、この関数はかなり明瞭でした。
重要なのは、
if (DAT_1432437cc <= param_2) {
    ...
}
と、
lVar3 = (longlong)(int)param_2 * 0x78 + DAT_1432437c0;
です。
つまり、
DAT_1432437c0
を先頭とする、
1エントリ 0x78 bytes のmodifier metadata配列
があります。
param_2 が配列index。
そして、
*(uint *)(lVar3 + 0x6c) = param_2;
*(undefined4 *)(lVar3 + 0x70) = param_1;
などを書き込みます。
Power Projectionの場合、
param_1 = 0x4d99
param_2 = 0x75
でした。
つまり、
0x4d99
と、
0x75
には別々の役割がある。
11. FUN_140539930 でその関係が確定的になる
FUN_14053b840 の最後に、
plVar2 =
    FUN_140539930(
        &DAT_142fa4b20,
        local_30,
        local_res8
    );

*(uint *)(*plVar2 + 0x14) = param_2;
がありました。
この FUN_140539930 をDecompilerすると、
冒頭で、
0xcbf29ce484222325
0x100000001b3
を使っていました。
これは64bit FNV-1aの定数。
さらに4バイトのkeyをハッシュして、
既存要素を探し、
無ければ新規nodeを作り、
unordered_map のbucketへ挿入する処理。
つまりほぼ、
unordered_map<uint32_t, uint32_t>
の find_or_insert / try_emplace 的処理です。
nodeには、
+0x10 : key
+0x14 : value
らしき構造が見えました。
したがって、
key   = 0x4d99
value = 0x75
が登録されている。
ここで、
0x4d99 = parser/token側の識別子
0x75 = modifier registry index

という理解が非常に強くなりました。
少なくとも、
0x4d99 → 0x75
というマッピングがunordered_mapに格納されていることはコード上明確です。
12. air_home_defence_factor も確認
Power Projectionの直前にあった、
MODIFIER_AIR_HOME_DEFENCE_FACTOR
も同じ登録処理を追跡。
すると、
MOV EDX,0x74
MOV ECX,dword ptr [RAX]
CALL FUN_14053b840
を発見。
したがって、
0x74 = air_home_defence_factor
0x75 = air_power_projection_factor
と確認。
つまりmodifier registry上で完全に隣接しています。
116 = air_home_defence_factor
117 = air_power_projection_factor
です。
13. modifier metadata tableを追った
modifier metadata array、
DAT_1432437c0
のXREFを表示すると大量に出ました。
100件近くあり、
ADD RCX,qword ptr [DAT_1432437c0]
などが大量。
一例として、
1410478fa
を追いました。
そこでは、
MOVSXD RAX,dword ptr [RBX + 0x500]
IMUL   RDX,RAX,0x78
ADD    RDX,qword ptr [DAT_1432437c0]
となっていました。
つまり、
descriptor =
    modifier_metadata[index];
という汎用処理。
これは実際のmodifier数値を扱うのではなく、
modifierの名前・定義・metadataを取得する処理
らしい、と判断。
このtableをXREF総当たりするのは効率が悪いので中止。
14. 116と117を同時に持つ関数を探す
ここで発想を変えました。
もし、
0x74 = Home Defence
0x75 = Power Projection
が対になって使われているなら、
同じ関数内に0x74と0x75の両方が現れる
可能性が高い。
Scalar Searchでは、
0x75 = 4149件
0x74 = 1948件
と、とても手で見られる量ではありませんでした。
そこでGhidra Java Scriptを作りました。
全関数の全命令を走査して、
0x74がある
AND
0x75がある
関数だけ表示。
結果は15関数。
FUN_1400810b0
FUN_140543c60
FUN_140f23c00
FUN_140f3bc90
FUN_14201bce0
FUN_142154450
FUN_1421c33e0
FUN_142227630
FUN_14224bbc0
FUN_1422c3990
FUN_1422caf50
FUN_1422d1280
FUN_142436c20
FUN_1424b4168
FUN_1424bd8e4
最初の2つはすでに、
FUN_1400810b0 → descriptor初期化
FUN_140543c60 → modifier登録
と分かっていたため除外。
そこで、
FUN_140f23c00
を調査。
これが大当たりでした。
15. FUN_140f23c00 で実際の航空処理へ到達
この関数にはかなり露骨に航空関連のRTTIが出ていました。
CAce
CAirBase
CShip
さらにソースパスまで残っていました。
C:\mnt\gsg\hoi4\hoi4_merged\hoi4\source\gamestate.h
や、
別関数では、
C:\mnt\gsg\hoi4\hoi4_merged\hoi4\source\airwing.cpp
も確認。
つまり完全にHOI4の航空隊処理です。
16. 最初に3つの航空statを取得
FUN_140f23c00 の冒頭。
plVar6 = FUN_140f25fc0(param_1,&local_res10,0x30);
*param_2 = *plVar6;

plVar6 = FUN_140f25fc0(param_1,&local_res10,0x2f,param_4);
param_2[1] = *plVar6;

plVar6 = FUN_140f25fc0(param_1,&local_res10,0x31,param_4);
param_2[2] = *plVar6;
つまり航空隊から、
stat 0x30
stat 0x2f
stat 0x31
という3能力値を取得。
当時はまだ、
AttackなのかDefenceなのかAgilityなのか？

は不明でした。
17. FUN_140f25fc0 の正体
Decompiler：
lVar4 = FUN_140f236a0(param_1 + 0x850,param_4);

if (lVar4 == 0) {
    uVar5 = *(undefined8 *)
        (param_1 + 0x390 + (longlong)param_3 * 8);
}
else {
    uVar5 = *(undefined8 *)
        (lVar4 + (longlong)param_3 * 8);
}

*param_2 = uVar5;
つまり、
statIndex × 8
で航空隊のstat arrayから値を取るgetter。
ミッション別値が存在すればそちらから取得。
要するに、
GetAirWingStat(
    AirWing,
    statIndex,
    MissionType
)
のような関数です。
これによって、
0x2f / 0x30 / 0x31
が実際に航空機statのindexだと分かりました。
18. 3能力値へ別々のfactorが入る
FUN_140f23c00 中盤には、
FUN_140542730(..., 0x6f);
FUN_140542730(..., 0x70);
FUN_140542730(..., 0x71);
がありました。
対応：
0x6f → lVar8
0x70 → uVar13
0x71 → lVar14
つまり、
stat 0x30 ← modifier 0x6f
stat 0x2f ← modifier 0x70
stat 0x31 ← modifier 0x71
という対応が存在。
ただしこの時点ではmodifier名が未判明。
19. そして決定的なHome / Projection分岐
その後、
任務対象Provinceを取得し、もう一方ではCAirBaseまたはCShipから基地・空母側の位置を解決します。
最終的に、
iVar2
iVar5
という2つの識別値を比較します。
後続の地理構造解析によって、どちらも対応するStrategic Regionの `+0x58` から取得される値だと特定できました。
決定的な部分：
if ((iVar2 != 0) && (iVar5 != 0)) {
    if (iVar2 == iVar5) {
        plVar6 =
            (longlong *)
            FUN_140542730(
                local_78 + 0x10,
                &local_res10,
                0x74
            );

        uVar9 = 0x74;
    }
    else {
        plVar6 =
            (longlong *)
            FUN_140542730(
                local_78 + 0x10,
                &local_res10,
                0x75
            );

        uVar9 = 0x75;
    }

    lVar3 = *plVar6;

    plVar6 =
        (longlong *)
        FUN_140542730(
            lVar7 + 0x10,
            &local_res10,
            uVar9
        );

    lVar8 = lVar8 + lVar3;
    uVar13 = uVar13 + *plVar6;
}
ここは非常に重要。
すでに、
0x74 = Home Defence
0x75 = Power Projection
と分かっています。
したがって、
iVar2 == iVar5
↓
同一Strategic Region
↓
air_home_defence_factor

iVar2 != iVar5
↓
別Strategic Region
↓
air_power_projection_factor
です。
20. 「home」の定義をStrategic Regionまで確定
分岐を発見した時点では、iVar2 / iVar5がState、Strategic Region、その他のlocation IDのどれなのか未確定でした。
そこでCProvinceの隣接フィールドと、State / Strategic RegionがProvinceを所属させる処理を追加で追跡しました。

まず `state.cpp` 由来の `FUN_1409c32d0` では、State定義に含まれるProvinceを取得した後、次の処理が行われています。

lVar1 = *(longlong *)(province + 0xc0);

if ((lVar1 != 0) && (lVar1 != param_1)) {
    // "Province ... is already added to state ..."
}

*(longlong *)(province + 0xc0) = param_1;

この関数の `param_1` はCState*なので、

CProvince + 0xC0 = CState*

が直接確認できます。したがって、航空計算で参照されていた隣の `+0xC8` はStateではありません。

次に `strategicregion.cpp` 由来の `FUN_140deb2a0` では、Provinceの `+0xC8` を検査し、別のオブジェクトが既に入っている場合に、

"Province has added another strategic region as theirs. Might be duplicates of this province in strategic regions templates."

という警告を出します。さらに警告中で、

*(undefined4 *)(*(longlong *)(province + 0xc8) + 0x58)

を「first region」の識別値として、

*(undefined4 *)(param_1 + 0x58)

を「overridden by」側のStrategic Region識別値として表示し、最後に、

*(longlong *)(province + 0xc8) = param_1;

と代入します。この `param_1` はCStrategicRegion*なので、

CProvince + 0xC8 = CStrategicRegion*

も直接確認できました。

これを `FUN_140f23c00` に戻して読むと、

targetProvince->strategicRegion->field_58

と、

baseProvince->strategicRegion->field_58

を比較していることになります。`CStrategicRegion + 0x58` の正式なC++メンバ名まではPDBなしでは分かりませんが、重複Strategic Region警告で各regionを識別する値として直接表示され、航空計算でも比較されているため、実質的なStrategic Region IDとみなせる識別値です。

一方の位置取得ではCAirBaseが明示的に扱われ、陸上基地でない場合にはCShipへのRTTI Castもあります。したがって空母艦載機も考慮され、最終条件は、

基地・空母側Provinceと任務対象Provinceが同一Strategic Region → Home Defence
基地・空母側Provinceと任務対象Provinceが別Strategic Region → Power Projection

です。これで「何をもってhomeと判定するのか」という最大の未解決点は解消しました。
21. 最大のポイント：Power Projectionは何に加算されているか
さっきのコードを見ると、
Power Projection / Home Defenceで取得したbonusは、
lVar8 += bonus;
uVar13 += bonus;
となっています。
しかし、
lVar14
には加算されません。
対応は、
lVar8   ← stat 0x30側
uVar13  ← stat 0x2f側
lVar14  ← stat 0x31側
でした。
したがって、
Power Projectionは3能力中2能力だけに効く。
残る問題は、
0x30 / 0x2f / 0x31 が具体的に何なのか？

でした。
22. 0x6f / 0x70 / 0x71 の名前を逆引き
ここで FUN_140543c60 に戻りました。
Java Scriptを使って、この巨大関数内だけで、
0x6f
0x70
0x71
を検索。
結果：
0x6f
14057bb98
14057bc75
14057bd52
14057be2f

0x70
14057bfb9
14057c096
14057c173
14057c250

0x71
14057c7fb
14057c8d8
14057c9b5
14057ca92
各modifierについて同じ登録処理が4回ずつ存在していました。
23. 0x6f の正体
14057bb98 周辺を調査。
すると、
LEA RDX,
[s_MODIFIER_AIR_ATTACK_FACTOR_14271d768]
つまり、
MODIFIER_AIR_ATTACK_FACTOR
を発見。
したがって、
0x6f = AIR_ATTACK_FACTOR
が確定。
そして、
0x6f → lVar8
lVar8 → stat 0x30
なので、
stat 0x30 = Air Attack
も対応。
24. 0x70 の正体
次に、
14057bfb9
周辺。
発見：
LEA RDX,
[s_MODIFIER_AIR_DEFENCE_FACTOR_14271d788]
つまり、
0x70 = AIR_DEFENCE_FACTOR
確定。
したがって、
stat 0x2f = Air Defence
と対応。
25. 0x71 の正体
最後に、
14057c7fb
周辺。
発見：
LEA RDX,
[s_MODIFIER_AIR_AGILITY_FACTOR_14271d7c8]
つまり、
0x71 = AIR_AGILITY_FACTOR
確定。
したがって、
stat 0x31 = Agility
と対応。
26. これで3能力の対応が完全に揃った
最終対応：
stat index	modifier index	名前
0x30	0x6f	Air Attack
0x2f	0x70	Air Defence
0x31	0x71	Agility


そして、
PowerProjectionBonus → lVar8
PowerProjectionBonus → uVar13

PowerProjectionBonus → lVar14 には入らない
だったので、
結論
air_power_projection_factor
        ↓
Air Attack factor へ加算
Air Defence factor へ加算
Agility factor には加算されない
です。
27. 数式として見るとどうなっているか
FUN_140f23c00 の最後では、
lVar8 =
    ((lVar11 + lVar8) * *param_2)
    / 100000;

*param_2 = lVar8;
Air Defence側も、
lVar8 =
    ((lVar11 + uVar13) * param_2[1])
    / 100000;
Agility側も同じ形式。
したがって内部では、
100000 = 100%
的なfixed-point表現を使っている可能性が非常に高いです。
概念的には、
\[
\text{Final Air Attack}
=
\text{Base Air Attack}
\times
\text{Attack Factor}
\]\[
\text{Final Air Defence}
=
\text{Base Air Defence}
\times
\text{Defence Factor}
\]\[
\text{Final Agility}
=
\text{Base Agility}
\times
\text{Agility Factor}
\]という構造。
Power Projectionは、
最終値へ最後に独立して掛けるのではなく、Air Attack / Air Defence のfactor蓄積値へ加算されてから基礎statに掛かる
ように見えます。
つまり、概念的には、
Attack Factor =
通常Attack補正
+ その他補正
+ Power Projection

Defence Factor =
通常Defence補正
+ その他補正
+ Power Projection

Agility Factor =
通常Agility補正
+ その他補正
です。
したがって例えば、
air_power_projection_factor = +10%
なら、
「別枠で最終値×1.10」
というより、
Attack / Defenceの加算型factor群へ +10% を入れる
という理解の方がコードに近いです。
28. air_home_defence_factor も同じ能力へ効く
同じ分岐で、
if (iVar2 == iVar5)
    0x74
else
    0x75
という選択しかしていません。
選ばれた値を、
lVar8
uVar13
へ同じように加えます。
したがって、
air_home_defence_factor
についても、
Air Attack + Air Defence
への補正です。
Agilityには入りません。
違うのは、
同一Strategic Region → Home Defence
別Strategic Region → Power Projection
という条件側だけ。
29. 「航空優勢ポイントを増やす」という説について
今回の解析だけから言えるのは、
少なくとも air_power_projection_factor はAir Attack / Air Defenceの計算へ直接入っている。
したがって、
Power Projectionは単にAir Superiorityへの寄与ポイントだけを増やすmodifier

という説明では不十分です。
ただし、
別のコードでもPower Projectionが使われ、航空優勢ポイントにも何か効果がある

可能性まで今回完全に排除したわけではありません。
今回確認したのは、
Power Projectionが実際にAir AttackとAir Defenceへ入るコードが存在する
ということ。
なので、
「航空優勢ポイントだけを増やす」
という説は少なくとも否定できます。
30. 今回ほぼ確定した内部番号一覧
後でGhidraを再開するときのためにまとめると、
内容	Hex	Decimal
Air Attack factor	0x6f	111
Air Defence factor	0x70	112
Air Agility factor	0x71	113
Air Home Defence factor	0x74	116
Air Power Projection factor	0x75	117
air_power_projection_factor token/key	0x4d99	19865


さらに、
stat 0x30 → Air Attack
stat 0x2f → Air Defence
stat 0x31 → Agility
も確認。
31. 重要関数一覧
今回かなり使った関数も記録しておくと、
FUN_1400810b0
modifier descriptor群の初期化をしている巨大関数。
ここで、
air_home_defence_factor
air_power_projection_factor
などの小文字名とtokenが紐付けられる。
FUN_1401467f0
実質、
*param_1 = param_2;
return param_1;
だけ。
整数wrapperのようなもの。
FUN_142412a40
整数IDと文字列を持つdescriptor/objectの初期化。
概念的には、
descriptor.id = param_2;
descriptor.name = param_3;
。
FUN_140543c60
巨大なmodifier metadata登録関数。
ここで、
MODIFIER_AIR_ATTACK_FACTOR
MODIFIER_AIR_DEFENCE_FACTOR
MODIFIER_AIR_AGILITY_FACTOR
MODIFIER_AIR_HOME_DEFENCE_FACTOR
MODIFIER_AIR_POWER_PROJECTION_FACTOR
などが登録される。
FUN_14053b840
modifier metadata entryを登録。
DAT_1432437c0 + index × 0x78
へ1エントリ書き込む。
同時にtoken → modifier index mapにも登録。
FUN_140539930
std::unordered_map 相当。
4byte keyをFNV-1aでhash。
今回、
0x4d99 → 0x75
のmapping登録に使用。
FUN_140f25fc0
AirWingのstat getter。
base + statIndex * 8
から値を取得。
MissionType別値があればそちらを使用。
FUN_140542730
modifier indexからmodifier値を取得するgetter系関数。
今回、
0x6f
0x70
0x71
0x74
0x75
0x76
などを渡していました。
FUN_140f23c00
今回の最重要関数。
航空隊の、
Air Attack
Air Defence
Agility
を取得し、
通常modifier、
Home Defence / Power Projection、
その他modifierを適用して最終能力値を計算する。
ここに、
CAirBase
CShip
も登場。
任務対象側と基地・空母側のStrategic Region識別値を比較し、同一なら0x74、別なら0x75を選択する。
今回の効果と発動条件を結び付けた核心。
FUN_1409c32d0
`state.cpp` 由来のState所属構築処理。
既に別Stateへ所属するProvinceへの警告を出した後、
province + 0xC0 = thisState
相当の代入を行う。
これにより、
CProvince + 0xC0 = CState*
を直接確認。
FUN_140deb2a0
`strategicregion.cpp` 由来のStrategic Region所属構築処理。
Provinceが別Strategic Regionを既に持つ場合の警告を出し、旧・新regionの `+0x58` を識別値として表示した後、
province + 0xC8 = thisStrategicRegion
相当の代入を行う。
これにより、
CProvince + 0xC8 = CStrategicRegion*
を直接確認。
32. 今回まだ分かっていないこと
home判定そのものは、今回の追加追跡でStrategic Region比較まで確定しました。
残っている型情報上の未解決点は、
CStrategicRegion + 0x58
の正式なC++メンバ名です。
ただし、
Strategic Region重複所属警告で旧・新regionの識別に使われる
航空計算で対象側と基地・空母側の比較に使われる
という二つの直接的な用法があるため、Strategic Regionを識別する値だという役割は十分確認できています。
したがって未確認なのはソース上の正式名称であって、比較対象のオブジェクト種別や判定条件ではありません。

もう一つ残るのは、
air_power_projection_factor
がこのAir Attack / Defence計算以外にも使用されているか。
今回、
Air Attack / Defenceへ効く

ことは確認しました。
でも、
それ以外には絶対何もしていない

ところまで全exeを証明したわけではありません。
ここは区別しておいた方がいいです。
33. 今回の証拠レベル
最後に、どこまで断定できるかを分けるとこうです。
ほぼコード上確定
0x74 = air_home_defence_factor
0x75 = air_power_projection_factor
0x6f = AIR_ATTACK_FACTOR
0x70 = AIR_DEFENCE_FACTOR
0x71 = AIR_AGILITY_FACTOR
stat 0x30 = Air Attack
stat 0x2f = Air Defence
stat 0x31 = Agility
CProvince + 0xC0 = CState*
CProvince + 0xC8 = CStrategicRegion*
航空計算は任務対象側と基地・空母側のCStrategicRegion + 0x58を比較する
同一Strategic Regionならair_home_defence_factorを選ぶ
別Strategic Regionならair_power_projection_factorを選ぶ
Home Defence / Power Projectionの値はAir AttackとAir Defenceのfactorへ加算される
Agilityへは加算されない
非常に強いが名称を完全には確定していない
CStrategicRegion + 0x58は実質的なStrategic Region ID。
識別値としての役割はコードから明確ですが、PDBがないため正式なC++メンバ名そのものは未確認。
今回は未確認
Power ProjectionがAir Attack / Defence以外の別システムでも追加利用されるか。
Power Projectionが航空優勢ポイントにも別途影響するか。
34. 一言で今回の成果をまとめると
最初はWikiやコミュニティに、
air_power_projection_factorって結局何なんだ？

という状態だったものを、
文字列
↓
XREF
↓
descriptor
↓
token 0x4d99
↓
modifier registry
↓
modifier index 0x75
↓
Home Defence 0x74との対
↓
航空隊能力計算 FUN_140f23c00
↓
Air Attack / Defence / Agilityの内部index
↓
0x6f = Attack
0x70 = Defence
0x71 = Agility
↓
Power ProjectionはAttackとDefenceだけに加算
↓
state.cppのFUN_1409c32d0でCProvince + 0xC0 = CState*を確認
↓
strategicregion.cppのFUN_140deb2a0でCProvince + 0xC8 = CStrategicRegion*を確認
↓
Strategic Region + 0x58が比較対象の識別値だと確認
↓
同一Strategic RegionならHome Defence、別Strategic RegionならPower Projection
というところまで、hoi4.exeそのものを逆アセンブルして追跡したわけです。
なので今回の最終的な成果は、
air_home_defence_factorは基地・空母と任務対象が同一Strategic Regionにある場合、air_power_projection_factorは別Strategic Regionにある場合に選択され、どちらもAir AttackとAir Defenceのfactorへ同じ経路で加算される。Agilityには加算されない。

というところまで到達した、というのが一番重要です。
そして個人的には、FUN_140f23c00 で
if (iVar2 == iVar5) {
    ...0x74...
}
else {
    ...0x75...
}
を見つけ、その後 0x6f / 0x70 / 0x71 をそれぞれ Attack / Defence / Agilityまで逆引きし、さらにCProvinceの `+0xC0` / `+0xC8` の所属代入から比較対象をStrategic Regionまで確定できたところが、今回の解析で一番決定的な部分です。
