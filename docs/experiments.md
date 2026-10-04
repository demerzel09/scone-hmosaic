# 実験仕様・保存・評価の規約

基準: [RESEARCH_SPEC.md](../RESEARCH_SPEC.md) §21–§30・§37–§40。

## 実験の入口

実験は `experiments/specs/<experiment_id>.yaml` から開始する。実装予定の ExperimentSpec validator は、草案・未確定項目・必須項目不足を拒否する。現在は validator も実行器も存在しない。

初期の項目案を以下に示す。コードによる schema 実装時に確定するまで、v0 の設計契約として扱う。

| 項目 | 必要な内容 |
| --- | --- |
| `schema_version`, `id`, `status`, `stage`, `kind` | 形式、識別子、draft/ready、段階、integration/scientific 等 |
| `hypothesis` | 仮説 ID、問い。基盤確認は H1 の prerequisite と明記 |
| `inputs` | scenario、engine/version、データ・設定の識別と由来 |
| `baseline` | 比較対象と固定設定 |
| `change` | 一つの primary change、比較範囲 |
| `evaluation` | 固定課題、条件、分割、反復、許容差、終了条件 |
| `metrics` | 指標 ID と、実装済みの計算方法への参照 |
| `seeds` | 実行前に固定する seed の全一覧 |
| `success` | 実行前に判定できる機械可読の条件 |
| `artifacts` | 保存先と必須成果物 |

草案にある `null` は未決定を表す。実行時の推測や暗黙の default で補完しない。ready への変更前に、計算方法・成功条件・入力を埋め、validation を通す。

[exp_001_scone_reproducibility.yaml](../experiments/specs/exp_001_scone_reproducibility.yaml) は M1 の再現性確認の草案である。同一 seed の反復実行を初回実行と比較する。H1 を直接検証する実験ではない。

## 分割と条件の固定

- 同じ episode の隣接サンプルを無作為に train/test へ混在させない。
- episode 単位の分割と、条件全体を除く held-out 条件を区別する。
- 正規化、閾値、module 数や hyperparameter の選択は train/development のみで行う。
- pilot で成功基準を決める場合は、確認用評価集合を別に固定する。
- task ごとの sample 数の差が評価を支配しないよう、集計の重みを事前指定する。
- すべての比較に同じ評価 task、seed 一覧、budget、観測・行動条件を使う。

## run の保存

各反復は固有の `run_id` を持ち、既存ディレクトリを上書きしない。

```text
experiments/runs/<run_id>/
  config.yaml          # 解決済み設定
  experiment.yaml      # 実験仕様の snapshot
  git_commit.txt
  environment.txt      # Python、依存、SCONE、engine、OS 等
  metadata.json        # seed、時刻、入力、終了コード、由来
  trajectory.parquet   # または HDF5。SCONE 抽出時
  metrics.json         # コードが生成
  stdout.log
  stderr.log
  summary.md
  plots/
```

run 開始時に設定と環境を保存する。未コミット変更を使う場合は diff と入力内容の hash も保存し、commit hash だけで再現可能とはしない。時刻は timezone を含む形式で保存する。

失敗した run は `failed`、中断は `interrupted` と記録し、ログ、終了コード、既存成果物を残す。軌跡が生成できなかった場合に空の成功データを捏造しない。指標生成器が動ける場合は未測定値を `null` とし理由を記録する。生成器自体の失敗で `metrics.json` が欠けた場合は、それも summary に明記する。

大きな成果物の保存方法は初回実装で決める。Git に含めるか外部保存するかにかかわらず、失敗した run を含めて永続化し、所在・hash を追跡できるようにする。

## 指標の契約

指標値は実装した計算コードから生成する。report に数値を載せる場合は run の `metrics.json` と対応させる。

| 領域 | 指標と事前に決める事項 |
| --- | --- |
| 再現性 | 列・shape・時刻の一致、許容差内の軌跡等価。比較列、絶対/相対許容差、時間整列を固定 |
| discovery | next-state / controller error、held-out error、module 数、parameter 数、MDL-like score |
| 共有 | 既存 module activation / 全 activation。hard/soft の集計と「複数条件で使う」の定義を固定 |
| 適応 | 固定性能閾値までの episode 数、更新 parameter 数 / 全 parameter 数 |
| 保持 | `Performance_before - Performance_after`。大きいほど良い性能値へ向きを揃える |
| 未知検出 | precision、recall、false positive rate、latency。未知ラベルと検出閾値を独立に固定 |
| task | 速度誤差、転倒率、距離、代謝 cost、CoT、安定性。算出可能性と定義を model ごとに確認 |

seed ごとの結果と集計を併記する。未測定とゼロを区別する。失敗 seed の除外を隠さず、欠測の扱いを仕様に固定する。

## 継続学習の固定課題列

`flat → different speeds → uphill → downhill → slippery → perturbation → ankle weakness → flat 再訪`

仕様書 §22 の順序を使い、各学習段階で過去課題も評価する。学習条件と評価条件を分け、最後の flat 再訪だけで全課題の保持を判断しない。速度・傾斜・弱化率などの値は benchmark 仕様に事前固定する。

## テストと可視化

unit は schema、設定、保存、失敗処理などを検証する。integration は実 SCONE から標準軌跡までを確認する。scientific は小さな toy system で broad な期待挙動を検証し、特定の浮動小数値を正解として研究を固定しない。

M1 の可視化は軌跡、接触、筋刺激などの抽出確認に限定する。discovery 以降は assignment、transition matrix、予測誤差、task ごとの共有を追加する。継続学習と未知検出の段階で性能曲線、forgetting、novelty score を追加する。

## 分析から次の判断へ

report は問い、baseline、固定した評価条件、全 seed の結果、成功基準との照合、限界、次の判断を含める。重要な判断は ADR に experiment/run ID を添えて残す。観測後に metric や seed を変える場合は新しい実験 ID を作る。
