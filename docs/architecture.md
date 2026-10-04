# 構成と責務

基準: [RESEARCH_SPEC.md](../RESEARCH_SPEC.md) §6–§20・§26–§28・§35–§40。

この文書は設計案であり、以下の component は未実装である。現在は文書と YAML 草案のみを置く。

## 最初のデータの流れ

```text
ExperimentSpec (YAML)
  → 入力検証・設定解決
  → run 記録開始
  → SCONE 実行
  → 生出力・終了状態・stdout/stderr 保存
  → trajectory parser
  → 標準データ・metadata・schema 検証
  → コードによる指標生成・最小可視化
  → report・次の判断
```

SCONE wrapper の実行方式は実 API と配布物の確認後に選ぶ。コマンドやファイル形式を推測して実装しない。失敗した run も保存する。raw output と標準データの対応を追跡できるようにする。

## component の境界

| 将来の配置 | 責務 | 初回の範囲 |
| --- | --- | --- |
| `src/simulator/` | SCONE 実行、engine/version/seed の記録、プロセス失敗処理 | 対象 |
| `src/dataset/` | 軌跡抽出、schema、feature order、transition 構築 | 対象 |
| `src/harness/` | ExperimentSpec、runner、run 保存、report | 薄い最小構成のみ |
| `src/evaluation/` | 指標の計算、可視化 | 再現性と軌跡確認のみ |
| `src/discovery/` | 局所線形 model、境界候補、共同 discovery | 次段階 |
| `src/modules/` | ForwardModel、Controller、MotorModule | 証拠を得てから |
| `src/mosaic/` | responsibility、soft/hard gating、runtime | 共有の確認後 |
| `src/hierarchy/` | activation sequence と上位構造 | 前段の成立後 |
| `src/snn/`, `src/plasticity/` | SNN と局所学習則 | 最後の段階 |

汎用 AutoML 基盤は作らない。ExperimentSpec、ExperimentRunner、MetricCollector、ResultStore、ReportGenerator は本研究の再現・比較・再開に必要な範囲に限定する。

## 予測・制御 module

`M_i = (F_i, C_i)` とし、`F_i(s_t, a_t) → predicted s_(t+1)`、`C_i(s_t) → predicted a_t` を持つ。初期 forward model は `A_i s_t + B_i a_t + c_i` とする。

discovery の目的は、正規化した予測誤差、controller 再現誤差、module 数の penalty を合わせた `L = E_pred + α E_ctrl + β N_modules` の比較である。厳密な description length を実装するまでは「MDL-like」と表記する。

接触、地面反力、筋活動、予測・制御誤差の変化を境界候補とする。時間区間の検出だけで共有 module を得たとしない。複数 episode・条件の局所モデルを共有する段階が必要である。

## MOSAIC の因果関係

仕様書 §12 の `P(M_i | s_t, a_t, s_(t+1))` は、遷移を観測した後の posterior responsibility である。`s_(t+1)` を使って同じ時刻の `a_t` を決めると未来情報を使ってしまう。

runtime では、現在までの履歴と `s_t` から得る事前 gate `g_(i,t)` で `a_t = Σ_i g_(i,t) C_i(s_t)` を決める。遷移観測後に posterior responsibility を計算し、次の gate と学習へ渡す。具体的推定法は runtime の実験仕様で決める。

soft gate と hard gate を同じ評価条件で比較可能にする。重みの正規化、行動範囲、数値異常時の扱いをテストする。

## 未知条件と局所更新

全 module の error が継続して大きい場合も、即座に新 module を作らない。

`既存 module の再利用 → 再結合 → 局所適応 → error 再評価 → 新 module の採否`

という順序を守る。新 module は複雑さ penalty を上回る改善がある場合に限る。保持 module の版と、どの parameter が更新されたかを記録する。

階層は下位 module の activation sequence の反復から検討する。support、swing などの意味分類を先にハードコードしない。SNN は通常モデルで成立した predictor/controller を段階的に置換する。

## repository の配置

現在のファイルは [readme.md](../readme.md) から参照する。将来、仕様書 §26 に沿って次の配置を追加する。

```text
configs/{scone,tasks,discovery,mosaic,plasticity}/
src/{simulator,dataset,discovery,modules,mosaic,hierarchy,snn,plasticity,evaluation,harness}/
scripts/
experiments/specs/       # 実験入口
experiments/runs/        # run 成果物
experiments/reports/     # 複数 run をまとめた分析
tests/{unit,integration,scientific}/
docs/decisions/
docs/status/
```

未実装領域の空クラスや仮の学習則は作らない。Python の packaging 方法、依存バージョン、SCONE 設定は初回実装で確定する。
