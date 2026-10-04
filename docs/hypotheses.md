# 仮説と検証条件

基準: [RESEARCH_SPEC.md](../RESEARCH_SPEC.md) §4・§21–§25・§42。

すべて未検証の研究仮説である。下表は評価項目の設計であり、結果や数値閾値の事前確定を意味しない。各実験で baseline、分割、seed、成功基準を実行前に固定する。

| ID | 仮説 | 必要な証拠 | 支持されない場合 |
| --- | --- | --- | --- |
| H1 | 異なる SCONE 運動に少数の共有予測・制御モジュールがある | 複数条件と held-out 条件での予測・制御再現、共有率、複雑さ。global model と条件別 model との比較 | trajectory ごとにほぼ独立 module が必要なら共有仮説は弱い |
| H2 | 新条件の一部は既存 module の再結合で成立する | module の重みを保存したまま組合せを変える閉ループ評価。新規学習との性能・必要 episode 数の比較 | 組合せだけでは成立せず大幅な再学習が必要なら、対象条件での再結合仮説は支持されない |
| H3 | 限定された module の更新で適応できる | 更新 parameter 比率、適応速度、既知課題の保持。全体更新や適応なしとの比較 | 多数 module や全 parameter の更新が必要なら局所性の主張は支持されない |
| H4 | module 保存が単一 policy より忘却を抑える | 固定課題列、各段階の過去課題評価、同じ計算・学習予算での forgetting 比較 | modular architecture でも保持が改善しない場合、構造だけでは頑健性を説明できない |
| H5 | forward prediction error が未知条件の検出に使える | 独立した既知・未知ラベル、precision/recall、誤警報率、検出遅延 | error が未知条件と対応しなければ、この信号だけでは不十分 |
| H6 | module の適応を BP なしの局所可塑性で実現できる | 同じ module 構造の gradient baseline と比較し、局所則の適応・保持・必要データ量を測る | gradient MOSAIC は成立し局所則だけが失敗するなら、構造と学習則の結論を分ける |

## H1 の最小評価

匿名 ID の予測・制御 model を複数条件で共同学習する。最初は Local Linear Model + Change Point Detection を使い、K = 1, 2, 4, 8, 16 を比較する。

forward model のみを評価する単一軌跡実験は、実装と局所力学の preliminary check と位置付ける。H1 の Go 判定には controller 再現、条件間共有、held-out 条件も必要である。

状態と行動の誤差は単位や次元に依存するため、訓練集合で決めた正規化と重みを使う。共有率が高くても予測・制御精度が不十分なら成功としない。複雑さの評価では module 数に加えて総 parameter 数を記録する。

## 効果を分離する比較

| 系列 | 構成 | 分離したい効果 |
| --- | --- | --- |
| A | Monolithic MLP + PPO / BP | 単一 policy の基準 |
| B | Modular Mixture-of-Experts + BP | module 構造の寄与 |
| C | MOSAIC + gradient learning | prediction による選択の寄与 |
| D | Modular controller + local learning | 局所学習の寄与 |
| E | Bio-MOSAIC + SNN + local plasticity | SNN と局所可塑性を含む方式 |

同じ観測・行動、課題、学習・評価 budget を明示する。単純な PPO 対 Bio-MOSAIC だけで modularity と非 BP の効果を結論しない。A–C は明示した gradient baseline として扱い、E に BP を混入させない。

## 将来の ablation

forward model、responsibility、module preservation、hierarchy、novelty detection、local adaptation、eligibility trace、local plasticity をそれぞれ外した条件を検討する。各実験の primary change は一つとし、複数要因の変更を同じ効果として扱わない。

## 結論の残し方

成功基準を満たした場合のみ「対象条件で支持された」と記載する。未実行、失敗、判定不能を区別する。No-Go と否定的結果も run、report、ADR に残し、観測後の基準変更は新しい実験仕様にする。
