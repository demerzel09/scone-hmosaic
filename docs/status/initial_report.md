# 初期文書整備の報告

日付: 2026-10-04（Asia/Tokyo）。対象: 仕様書の把握と初期文書の整備。

## 整備したもの

- `readme.md`: 研究目的、現在地、文書の入口、最初の実装と停止条件。
- `AGENTS.md`: 科学的優先順位、実験規律、Bio-MOSAIC 制約、証拠と報告。
- `docs/research_plan.md`: 段階、番号の対応、M1 の受け入れ条件、Go / No-Go。
- `docs/hypotheses.md`: H1–H6、比較対象、支持されない条件。
- `docs/architecture.md`: component の責務と runtime の因果関係。
- `docs/experiments.md`: YAML の契約案、分割、run 保存、指標、評価。
- `docs/data_contract.md`: 標準軌跡、transition、metadata、検証。
- `docs/decisions/0001-research-scope-and-gates.md`: 初期判断。
- `experiments/specs/exp_001_scone_reproducibility.yaml`: 未実行の再現性実験草案。

## 確認した開始時の状態

- branch は `main`、remote は `https://github.com/demerzel09/scone-hmosaic.git`。
- local に commit はなく、remote の branch 一覧も空だった。
- 作業ツリーのファイルは未追跡の `RESEARCH_SPEC.md` のみだった。
- Python は 3.10.12。
- `scone` と `hyfydy` は現在の `PATH` で見つからなかった。別の場所への導入の有無は未確認。

## 未実装・未実行

Python project、ExperimentSpec の validator、result storage、SCONE runner、parser、schema validator、plot、test は未実装。実 SCONE の歩行、dataset 生成、module discovery は実行していない。科学的指標と判定結果はない。

この報告は Milestone 1 の完了報告ではない。`docs/status/milestone_1_report.md` は、実装と実行を行った後に作成する。

## 文書の検証

- Markdown の相対リンク先と code fence の対応を確認した。
- YAML の構文、重複 key、ID、seed 一覧、metric 参照、未確定項目の保持を確認した。
- `RESEARCH_SPEC.md` の SHA-256 が開始時と一致し、原本を変更していないことを確認した。
- コードのテストと実 SCONE の検証は、実装がないため実行していない。

## 次に確認すること

SCONE/Hyfydy の導入方法、使用可能なエンジン、version と seed の指定・取得、実行 API、raw 出力の時刻・列・単位、歩行 scenario とモデルの依存関係を確認する。再現性許容差は独立した pilot の後、確認用実験の実行前に固定する。

M1 の実装後に停止し、次の dataset/discovery 段階の推奨を報告する。初回の discovery は局所線形モデルを使う。SNN や局所可塑性の先行実装は行わない。
