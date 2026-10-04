# SCONE × Hierarchical MOSAIC × Local Plasticity

筋骨格シミュレーション上で、保存された予測・制御モジュールの選択、再結合、局所適応が、継続的な運動学習を支えられるかを研究するリポジトリです。

研究仕様の基準は [RESEARCH_SPEC.md](RESEARCH_SPEC.md) です。SCONE による最適化は、身体に適した運動構造を得る長期探索の代理として扱います。実際の生物進化を再現したという主張はしません。

## 検証する問い

最初に検証するのは、**複数条件の SCONE 運動を、少数の共有可能な予測・制御モジュールで表現できるか**です。

各モジュールは `M_i = (F_i, C_i)` として、状態遷移を予測する forward model と、筋刺激を出す controller を組にします。単なる時間区間や、あらかじめ名前を付けた歩行フェーズはモジュールの定義にしません。

共有構造が確認できた後に、再利用、MOSAIC による選択、未知状態検出、局所適応、階層化を検証します。SNN と局所可塑性はその後の段階です。単一課題のスコアだけでなく、適応速度、過去技能の保持、身体・環境の変化への応答を評価します。

## 現在の状態

2026-10-04 時点では、研究開始のための文書を整備した段階です。

- 研究計画、仮説、構成、実験規約、データ契約、判断記録を作成しました。
- 最初の再現性実験の YAML 草案を用意しました。
- Python パッケージ、SCONE runner、データ変換、テスト、実験実行は未実装です。
- 実験結果、モジュール共有の証拠、Bio-MOSAIC の性能評価はまだありません。

詳細は [初期整備の報告](docs/status/initial_report.md) に記録しています。

## 文書の入口

| 文書 | 内容 |
| --- | --- |
| [RESEARCH_SPEC.md](RESEARCH_SPEC.md) | 研究目的、制約、要求事項の原本 |
| [AGENTS.md](AGENTS.md) | エージェントが守る研究・実装規約 |
| [研究計画](docs/research_plan.md) | 段階、依存関係、Go / No-Go、最初の作業 |
| [仮説](docs/hypotheses.md) | H1–H6、必要な証拠、棄却条件 |
| [構成](docs/architecture.md) | モジュール境界、データの流れ、将来の構成 |
| [実験規約](docs/experiments.md) | YAML の項目、評価設計、run の保存、報告 |
| [データ契約](docs/data_contract.md) | 軌跡、状態・行動・遷移、メタデータ、検証 |
| [初期判断 ADR](docs/decisions/0001-research-scope-and-gates.md) | 文書整備の範囲と研究ゲート |
| [最初の実験草案](experiments/specs/exp_001_scone_reproducibility.yaml) | SCONE 実行・抽出の再現性確認。未実行 |

## 最初の実装

次の実装対象は、仕様書 §32・§43 の **SCONE 実行と軌跡抽出の再現可能な基盤**です。

1. SCONE の導入方法、バージョン、実行 API、使用エンジン、既知の歩行シナリオを確認する。
2. Python プロジェクト、ExperimentSpec の検証、run 保存を整える。
3. シナリオと seed を指定できる runner を作り、失敗時もログと終了コードを保存する。
4. 軌跡を Parquet または HDF5 に変換し、スキーマと同一 seed の再現性を検証する。
5. 最小可視化とテストを実行し、`docs/status/milestone_1_report.md` を作成する。

仕様書で要求される次のコマンドは、**実装後の受け入れ目標**です。現在はスクリプトとシナリオが存在せず、実行できません。

```bash
python scripts/run_scone.py \
    --scenario configs/scone/walk.scone \
    --seed 0 \
    --output experiments/runs/test_walk
```

SCONE の導入済み実体やライセンス、シナリオの配布条件は未確認です。確認できるまで、導入コマンドや出力形式を確定しません。

## 研究の進め方

研究ループは `仮説 → YAML 実験仕様 → 実装 → テスト → 実行 → 指標 → 分析 → 判断 → 次の実験` とします。成功基準は実行前に固定し、複数 seed、明示した baseline、固定評価条件を使います。失敗した run も保存し、指標値はコードから生成します。

Bio-MOSAIC と明記した実験では、backpropagation と勾配によるシナプス重み更新を禁止します。勾配法は明示した比較用 baseline に限ります。

Milestone 1 の実装・報告を終えたら停止し、次の実装段階へはユーザーの明示的な承認を得て進みます。複数条件での共有モジュールが成立しない場合も、原因を報告して人間のレビューを待ちます。

## 文書の更新

研究目的や制約を変える場合は `RESEARCH_SPEC.md` と ADR に明示します。下位文書は仕様書の具体化であり、仕様書を暗黙に上書きしません。結果報告には実験 ID、run ID、コードの commit、判定基準と証拠を含めます。
