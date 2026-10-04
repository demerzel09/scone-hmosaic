# SCONE × Hierarchical MOSAIC × Local Plasticity
## Reusable Motor Modules for Continual Motor Learning without Backpropagation

# 0. この文書の目的

このリポジトリでは、

- SCONE / Hyfydy による筋骨格シミュレーション
- MOSAIC / Hierarchical MOSAIC に基づく予測・制御モジュール
- SNN
- STDP / Reward-modulated STDP
- eligibility trace
- homeostatic / inhibitory plasticity

を組み合わせ、

「巨大な単一ニューラルネットワークを Backpropagation で再最適化する」

のとは異なる、生物的な運動獲得モデルを研究する。

最終目標は単に

> BPなしで歩行を学習できるか

を示すことではない。

中心仮説は、

> 進化的探索によって獲得された少数の予測‐制御モジュールを保存し、
> 状況に応じて選択・再結合・局所修正する方式は、
> end-to-end BP による単一ポリシー学習とは異なる形で、
> 継続学習、技能再利用、未知状態検出、少量適応に優れる可能性がある

というものである。

この仮説を、SCONE の筋骨格物理系上で段階的に検証する。


# 1. 最重要設計原則

いきなり

SCONE + HMOSAIC + SNN + STDP

を全部実装してはいけない。

以下の順に仮説を分離して検証する。

1. SCONE の複数最適運動に共有可能なモジュール構造が存在するか
2. 発見したモジュールを再利用して別条件の運動を構築できるか
3. MOSAIC による選択・組合せ・未知状態検出が機能するか
4. オンライン局所適応が有効か
5. 階層構造が有効か
6. 最後に通常NNをSNNへ置換する
7. Backpropagation を完全に外して局所可塑性だけで適応する

最初の Go / No-Go は、

> 複数のSCONE最適行動を少数の共有予測‐制御モジュールとして
> 圧縮できるか

である。

これが成立しない場合、
SNN実装へ進む前に研究仮説を再検討する。


# 2. 研究背景

筋骨格系には、

- 骨格形状
- 質量分布
- 関節可動域
- 筋の力長特性
- 筋の力速度特性
- 腱
- 重力
- 地面接触
- 摩擦
- 慣性

など強い物理的制約が存在する。

そのため、

- 前進速度
- 転倒回避
- エネルギー消費
- 安定性

など比較的高レベルな目的関数からでも、
自然に近い周期運動が生成され得る。

SCONE はそのような筋骨格モデルに対して
CMA-ES 等によるシミュレーションベース最適化を行える。

本研究では SCONE 最適化を、

「子供自身の学習」

とはみなさない。

SCONEは、

> 長い時間スケールの探索によって身体に適した運動構造が
> あらかじめ獲得されている状態

を人工的に生成するために使用する。

概念的には、

SCONE optimization ≈ evolutionary search proxy

とする。

ただし、これは実際の生物進化を忠実に再現しているという主張ではない。
あくまで計算論的役割の対応である。


# 3. 生物学的仮説

本研究では以下を仮定する。

Biological Learning
≈
Selection
+ Preservation
+ Composition
+ Local Adaptation

すなわち生物は、新しい状況に遭遇するたびに
巨大なニューラルネットワーク全体を再最適化するのではなく、

1. 既存の運動モードを保存する
2. 状況に適したものを選択する
3. 複数モードを組み合わせる
4. 必要部分のみ局所修正する
5. 既存モデルで説明不能な状況だけ新モジュールを形成する

と仮定する。


# 4. 研究仮説

## H1: Reusable Motor Module Hypothesis

異なるSCONE最適運動には、
少数の再利用可能な予測‐制御モジュールが存在する。


## H2: Composition Hypothesis

新しい運動条件の一部は、

existing modules + new composition

だけで生成でき、
新しいコントローラをゼロから学習する必要がない。


## H3: Local Adaptation Hypothesis

新しい環境への適応は、

M_k -> M'_k

のような限定されたモジュール更新で成立する。


## H4: Continual Learning Hypothesis

モジュールを保存することで、
単一 end-to-end policy より catastrophic forgetting が小さくなる。


## H5: Novelty Detection Hypothesis

forward model の prediction error から、

「現在の身体・環境条件は既知モデルでは説明できない」

ことを検出できる。


## H6: Local Plasticity Hypothesis

適切なモジュール構造が存在すれば、
各モジュールの適応は BP を使わず、

- STDP
- R-STDP
- eligibility trace
- homeostasis

等の局所則で実現できる。


# 5. SCONE の役割

SCONE から単一の完成歩行を取得し、
その時系列をニューラルネットへ模倣させることを主目的にしない。

SCONE は多数の条件で、
身体力学的に成立する運動解を生成するために使う。

例えば、

speed:
- 0.6
- 0.8
- 1.0
- 1.2
- 1.4
- 1.6 m/s

slope:
- -10 deg
- -5 deg
- 0
- +5 deg
- +10 deg

body conditions:
- normal
- weak ankle
- weak knee
- heavy torso

disturbance:
- none
- forward push
- lateral push

などを生成する。


# 6. SCONEから保存するデータ

各時刻について最低限、

state s_t
action a_t
next_state s_{t+1}

を保存する。

state候補:

- joint positions
- joint velocities
- muscle fiber lengths
- muscle fiber velocities
- muscle forces
- body segment poses
- center of mass
- contact states
- ground reaction forces

action:

- muscle excitations / stimulations

metadata:

- task id
- environment id
- body parameters
- optimization seed
- SCONE scenario
- simulation engine
- optimization fitness
- episode id
- timestep

保存形式は CSV ではなく、

Parquet または HDF5

を基本とする。


# 7. Motor Module の定義

単なる時間区間として module を定義しない。

module は、

「限定された状態領域において再利用可能な予測‐制御則」

とする。

各 module M_i は、

M_i = (F_i, C_i)

で表す。

F_i:
forward model

F_i(s_t, a_t) -> s_hat_{t+1}

C_i:
controller

C_i(s_t) -> a_hat_t


# 8. Module Discovery

SCONEデータから module を自動発見する。

forward prediction error:

E_pred =
|| s_{t+1} - F_i(s_t, a_t) ||^2

control reproduction error:

E_ctrl =
|| a_t - C_i(s_t) ||^2

module complexity penalty:

E_complexity =
beta * N_modules

総目的:

L =
E_pred
+ alpha * E_ctrl
+ beta * N_modules

モジュールを増やせば誤差は小さくなるため、
module数そのものにコストを課す。

目標は、

「できるだけ少数のmoduleで、
複数環境・複数動作を説明する」

ことである。

Minimum Description Length 的な考え方を使用する。


# 9. 最初の Module Discovery 実装

最初からニューラルネットを使わない。

まず以下を比較する。

1. Local Linear Model
2. Gaussian Mixture + Local Dynamics
3. Switching Linear Dynamical System
4. HMM / Switching State Space Model

最初のbaselineは、

Local Linear Model + Change Point Detection

でよい。

目的は高精度化ではなく、

「局所力学モデルの切替として歩行を圧縮できるか」

を検証すること。


# 10. Module Boundary

時間位置だけで module を分割しない。

以下の変化を利用する。

- forward prediction error increase
- controller error increase
- contact state transition
- ground reaction force transition
- muscle activation pattern change

特に接地・離地は運動方程式そのものが変化するため、
自然な module boundary 候補となる。


# 11. Multi-condition Module Discovery

単一歩行だけから module を発見しない。

複数条件をまとめて学習し、

条件を越えて繰り返し利用される構造を module とみなす。

期待される例:

M1:
weight acceptance

M2:
stance stabilization

M3:
push-off

M4:
swing advancement

M5:
landing stabilization

条件固有:

M6:
strong uphill propulsion

M7:
high-speed swing

M8:
perturbation recovery

ただし、この名前は人間が先に付けない。

最初は module_001 のような匿名 module とし、
挙動解析後に意味を解釈する。


# 12. MOSAIC Runtime

各 module:

M_i = (F_i, C_i)

forward prediction error:

e_i =
|| s_{t+1} - F_i(s_t, a_t) ||

responsibility:

lambda_i =
P(M_i | s_t, a_t, s_{t+1})

sum_i lambda_i = 1

action:

a_t =
sum_i lambda_i C_i(s_t)

soft gating と hard gating の両方を実装可能にする。


# 13. Novelty Detection

すべての module について、

e_i > epsilon

が一定時間継続した場合、

「既知のどのモデルも現在の状態遷移を説明できない」

と判断する。

ただし即座に module を追加しない。

順序:

1. known module reuse
2. module recombination
3. local adaptation
4. prediction error re-evaluation
5. new module creation

とする。


# 14. New Module Creation

単純な

error > threshold -> create module

は禁止する。

新 module 作成条件:

1. prediction error が一定時間以上高い
2. 既存 module の組合せで改善しない
3. 既存 module の local adaptation でも改善しない
4. 新 module が complexity penalty 以上の改善を与える

概念的には、

Delta Error Reduction
>
beta * Delta Complexity

の場合のみ新moduleを追加する。


# 15. Hierarchical MOSAIC

module を単一階層に限定しない。

想定される構造:

Layer 0:
reflex / muscle control

Layer 1:
motor primitive

Layer 2:
skill

Layer 3:
behavior / strategy

例:

Layer 0:
- stretch reflex
- reciprocal inhibition
- force feedback

Layer 1:
- support
- push-off
- swing
- landing

Layer 2:
- walking
- running
- stair climbing

Layer 3:
- move forward
- accelerate
- avoid obstacle

ただし、
この分類をコードへハードコードしない。


# 16. Hierarchy Discovery

下位 module の activation sequence を保存する。

例:

A C B A C B A C B

複数タスクで

A C B

が繰り返し現れるなら、

H1 = Sequence(A, C, B)

として上位 module 候補とする。

最初は、

- n-gram
- frequent sequence mining

程度で実装する。

後から、

- MDL
- Bayesian non-parametric model
- hierarchical state-space model

を検討する。


# 17. 完成歩行を事前に与えない実験

重要な実験として、
SCONEで完成歩行だけを最適化するのではなく、

以下の局所能力を別々に生成する。

- body support
- forward propulsion
- swing leg advancement
- landing stabilization
- posture recovery

その後 HMOSAIC が、

support
-> propulsion
-> swing
-> landing
-> support

という循環を形成することで、
完成歩行が創発するか調べる。

これが成立すれば、

「歩行そのものを教師として与えた」

という説明を避けられる。


# 18. 通常NNによる実証を先に行う

SNNを最初から使わない。

まず、

small MLP / local model

を使用して、

- module discovery
- MOSAIC gating
- module reuse
- continual learning
- hierarchy

が成立することを確認する。

理由:

失敗時に、

- SNN表現
- learning rule
- module structure
- gating
- simulator

のどこが原因か分離できなくなるため。


# 19. SNNへの置換

通常モデルで成立後、
各 controller / predictor を SNN へ置換する。

初期候補:

1. LIF
2. Adaptive LIF
3. Izhikevich

最初は LIF を採用する。

研究上重要なのは neuron model の詳細ではなく、

「局所学習だけで適応可能か」

である。


# 20. Local Plasticity

段階的に追加する。

## Stage A

STDP

局所 pre/post timing correlation のみ。


## Stage B

Reward-modulated STDP

global reward R を追加。


## Stage C

Eligibility Trace

各 synapse に e_ij を保持する。


## Stage D

MOSAIC responsibility を追加する。

基本更新式:

Delta w_ij =
eta
* e_ij
* R
* lambda_k

意味:

- このシナプスが関与した
- 結果が有益だった
- このmoduleが現在の状態に責任を持っていた

場合にのみ更新する。


## Stage E

Homeostatic Plasticity

過剰発火・活動消失を防止する。


## Stage F

Inhibitory Plasticity

必要になった場合のみ追加する。


# 21. BP比較で注意すること

単純な

PPO vs Bio-MOSAIC

だけでは不十分。

なぜなら、

「モジュール構造の効果」

と

「BPを使わない効果」

が混ざるため。


最低限以下を比較する。


A:
Monolithic MLP
+ PPO / BP

B:
Modular Mixture-of-Experts
+ BP

C:
MOSAIC
+ gradient learning

D:
Modular controller
+ local learning

E:
Bio-MOSAIC
+ SNN
+ local plasticity


これにより、

modularity effect

と

non-BP learning effect

を分離する。


# 22. Continual Learning Benchmark

固定された順序で課題を与える。

Stage 1:
flat walking

Stage 2:
different walking speeds

Stage 3:
uphill

Stage 4:
downhill

Stage 5:
slippery ground

Stage 6:
external perturbation

Stage 7:
ankle weakness

Stage 8:
return to flat walking


Stage 8 で Stage 1 の能力が保持されているか確認する。


# 23. Metrics

## Task performance

- walking velocity error
- fall rate
- traveled distance
- metabolic cost
- cost of transport
- balance stability


## Adaptation speed

新環境で性能閾値へ達するまでの

N_episodes


## Retention

過去課題性能:

Forgetting_i =
Performance_before
-
Performance_after


## Module reuse

Reuse =
existing module activation
/
total module activation


## Locality

Locality =
number of modified parameters
/
total parameters


## Novelty detection

- precision
- recall
- false positive rate
- detection latency


## Structural complexity

- number of modules
- total module parameters
- hierarchy depth
- description length


# 24. Ablation Study

最低限以下を外して比較する。

- forward model
- responsibility signal
- module preservation
- module hierarchy
- novelty detection
- local adaptation
- eligibility trace
- local plasticity

各要素が本当に必要か検証する。


# 25. Failure Criteria

成功条件だけでなく仮説棄却条件も記録する。


## Failure A

異なるSCONE運動を説明するために、
ほぼ各trajectoryごとに別moduleが必要。

Interpretation:
reusable motor module hypothesis is weak.


## Failure B

modular architecture を使用しても
continual-learning retention が改善しない。

Interpretation:
modularity alone does not explain robustness.


## Failure C

Gradient MOSAIC は機能するが
Local-Plasticity MOSAIC は学習不能。

Interpretation:
module structure may be valid,
but proposed biological learning rule is insufficient.


## Failure D

prediction error が novelty detection と相関しない。

Interpretation:
forward-model error alone is insufficient as novelty signal.


# 26. Repository Structure

bio-mosaic-locomotion/
|
|-- README.md
|-- AGENTS.md
|-- pyproject.toml
|-- Makefile
|
|-- docs/
|   |-- research_plan.md
|   |-- hypotheses.md
|   |-- architecture.md
|   |-- experiments.md
|   `-- decisions/
|
|-- configs/
|   |-- scone/
|   |-- tasks/
|   |-- discovery/
|   |-- mosaic/
|   `-- plasticity/
|
|-- src/
|   |-- simulator/
|   |-- dataset/
|   |-- discovery/
|   |-- modules/
|   |-- mosaic/
|   |-- hierarchy/
|   |-- snn/
|   |-- plasticity/
|   |-- evaluation/
|   `-- harness/
|
|-- scripts/
|   |-- run_scone.py
|   |-- optimize_scone.py
|   |-- collect_dataset.py
|   |-- discover_modules.py
|   |-- run_mosaic.py
|   |-- train_biomosaic.py
|   `-- benchmark.py
|
|-- experiments/
|   |-- specs/
|   |-- runs/
|   `-- reports/
|
`-- tests/


# 27. Agentic Research Harness

この研究では、
AIエージェントに自由研究をさせない。

研究ループを以下に固定する。

Hypothesis
-> Experiment Specification
-> Implementation
-> Test
-> Execution
-> Metrics
-> Analysis
-> Decision
-> Next Experiment


Experiment の唯一の入口を YAML にする。


例:

id: exp_003_module_count

hypothesis:
  reusable local predictive-control modules
  can explain multiple locomotion conditions

baseline:
  config: configs/discovery/base.yaml

change:
  num_modules:
    - 2
    - 4
    - 8
    - 16

metrics:
  - prediction_error
  - control_error
  - mdl_score
  - cross_task_reuse

seeds:
  - 0
  - 1
  - 2
  - 3
  - 4

success:
  conditions:
    - mdl_score improves over one-module baseline
    - cross_task_reuse > 0.5


# 28. Run Artifact

各runについて、

experiments/runs/<run_id>/

を作成する。

必須ファイル:

config.yaml
experiment.yaml
git_commit.txt
environment.txt
metrics.json
stdout.log
stderr.log
summary.md
plots/


metrics.json は必ずコードから生成する。

LLMに数値を手書きさせない。


# 29. Research Decision Log

重要な研究判断は、

docs/decisions/

へADRとして保存する。


例:

# ADR-0003

## Observation

4 modules significantly reduce prediction error
compared with 2 modules.

8 modules provide only minor additional improvement.

## Decision

Use 4 modules as current default.

## Evidence

- exp_003
- exp_004

## Revisit if

cross-environment dataset exceeds 20 locomotion conditions.


# 30. AGENTS.md Rules

以下をAGENTS.mdに記述する。


# Research Agent Rules

This repository investigates whether reusable
predictive-control modules and local plasticity
can support continual motor learning without
backpropagation.

## Scientific priority

Do not optimize benchmark scores by silently
changing the scientific question.

## Experimental discipline

Every experiment must:

1. reference an explicit hypothesis,
2. define one primary experimental change,
3. use fixed evaluation tasks,
4. use multiple random seeds,
5. retain failed experiments,
6. generate machine-readable metrics,
7. compare against an explicit baseline.

## Bio-MOSAIC constraint

No backpropagation or gradient-based synaptic
weight update is allowed inside experiments
explicitly labeled Bio-MOSAIC.

Gradient methods are allowed only for
explicit baseline models.

## Implementation discipline

Before implementing a complex mechanism:

1. implement the smallest falsifiable version,
2. write tests,
3. run it on a minimal toy system,
4. only then integrate it with full SCONE.

## Reporting

Never claim an experiment supports a hypothesis
unless predefined success criteria are satisfied.

Negative results must be preserved.

Do not delete failed runs.

Do not change metrics after observing results
without creating a new experiment specification.

Do not change random seeds selectively.

Do not silently change benchmark conditions.


# 31. Development Roadmap

Milestone 1:
SCONE execution

Milestone 2:
Trajectory dataset generation

Milestone 3:
Single-trajectory module discovery

Milestone 4:
Cross-condition reusable module discovery

Milestone 5:
MOSAIC runtime

Milestone 6:
Module reuse and recombination

Milestone 7:
Novelty detection

Milestone 8:
Local module adaptation

Milestone 9:
Dynamic module creation

Milestone 10:
Hierarchical module discovery

Milestone 11:
SNN controller replacement

Milestone 12:
Local plasticity

Milestone 13:
Continual learning benchmark

Milestone 14:
Comparison with BP / PPO / modular BP baselines


# 32. Milestone 1: SCONE Integration

FIRST IMPLEMENTATION TASK.

Do not implement MOSAIC or SNN yet.

Goal:

Create a reproducible SCONE execution and
trajectory extraction layer.


Required work:

1. Create Python project structure.
2. Create SCONE command-line wrapper.
3. Support scenario path.
4. Support fixed random seed.
5. Record engine type.
6. Record SCONE version.
7. Execute one known walking scenario.
8. Extract trajectory.
9. Convert trajectory into standard dataset format.
10. Add reproducibility test.


Required command:

python scripts/run_scone.py \
    --scenario configs/scone/walk.scone \
    --seed 0 \
    --output experiments/runs/test_walk


Expected output:

experiments/runs/test_walk/
    config.yaml
    trajectory.parquet
    metadata.json
    stdout.log


trajectory.parquet should contain at least:

episode_id
time
joint_positions
joint_velocities
muscle_lengths
muscle_forces
muscle_excitations
contact_states


Acceptance criteria:

- identical seed produces equivalent trajectory
- output schema is validated
- failed SCONE process returns non-zero status
- stdout/stderr are preserved
- unit tests pass


# 33. Milestone 2: Minimal Module Discovery

Research question:

Can one optimized locomotion trajectory be
represented by a small number of local
predictive dynamics models?


DO NOT use neural networks initially.


Baseline:

piecewise linear forward dynamics.


Input:

(s_t, a_t, s_{t+1})


Model:

s_hat_{t+1}
=
A_i s_t
+
B_i a_t
+
c_i


Compare:

K = 1, 2, 4, 8, 16


Metrics:

- next-state prediction error
- held-out trajectory prediction error
- transition stability
- number of modules
- MDL-like score


Do not interpret modules biologically yet.


# 34. Milestone 3: Cross-condition Discovery

Generate trajectories for multiple:

- walking speeds
- slopes

Fit modules jointly.

Research question:

Do the same modules recur across different
locomotion conditions?


Primary metric:

cross-condition module reuse.


Important:

A successful result is NOT simply lower
prediction error.

A successful result requires both:

1. acceptable predictive accuracy
2. module reuse across conditions


# 35. Milestone 4: MOSAIC

After reusable modules are demonstrated,
implement responsibility-based module selection.


Required components:

ForwardModel

Controller

MotorModule

ResponsibilityEstimator

MosaicController


Suggested interface:


class ForwardModel:
    def predict(self, state, action):
        raise NotImplementedError


class Controller:
    def act(self, state):
        raise NotImplementedError


class MotorModule:
    def predict(self, state, action):
        ...

    def act(self, state):
        ...

    def prediction_error(self, transition):
        ...


class MosaicController:
    def responsibilities(self, state, transition=None):
        ...

    def act(self, state):
        ...

    def update(self, transition, reward=None):
        ...


# 36. Milestone 5+: Do Not Pre-implement

Do not create speculative implementations for:

- SNN
- STDP
- hierarchical module creation
- novelty-triggered module growth

until earlier milestones have experimental evidence.

Create interfaces if necessary,
but leave implementations minimal.


# 37. Testing Strategy

Tests must be divided into:

tests/unit/
tests/integration/
tests/scientific/


Unit tests:

- dataset schema
- module prediction
- responsibility normalization
- config loading


Integration tests:

- SCONE -> trajectory
- trajectory -> module discovery
- module -> prediction


Scientific tests:

Example:

test_module_discovery_beats_single_model()

This test should verify broad expected behavior,
not exact floating-point values.


# 38. Reproducibility

Every experiment must capture:

- git commit hash
- Python version
- dependency versions
- SCONE version
- simulation engine
- operating system
- random seeds
- experiment configuration
- timestamp


# 39. Visualization

Generate at least:

1. SCONE motion trajectory plots
2. module assignment over time
3. module transition matrix
4. prediction error over time
5. module reuse per task
6. task performance over learning
7. forgetting curves
8. novelty score over time


For module visualization:

time
------------------------------------------------>
M1 M1 M1 M2 M2 M3 M3 M1 ...

should be visually inspectable.


# 40. Research Harness Philosophy

Do not build a large generic AutoML platform.

Build a thin research harness specific to
this project.

Core abstraction:

ExperimentSpec
ExperimentRunner
MetricCollector
ResultStore
ReportGenerator


The harness should make experiments:

- reproducible
- comparable
- resumable
- machine-readable
- reviewable by humans
- executable by Codex / Claude Code


# 41. Scientific Guardrail

The following statement must guide development:

The purpose of this project is not to make
Bio-MOSAIC beat PPO at a single locomotion task.

The important question is whether a system based on:

- preserved modules
- prediction-based selection
- reuse
- composition
- local adaptation

offers advantages under:

- continual learning
- changing body dynamics
- environmental changes
- recovery of previous skills
- low-data adaptation
- novelty detection.


# 42. First Go / No-Go Decision

Before implementing SNN or biological plasticity,
produce an experimental report answering:

> Can multiple SCONE-generated locomotion behaviors
> be compressed into a small set of reusable
> predictive-control modules?

Required evidence:

- multiple locomotion conditions
- held-out conditions
- prediction metrics
- controller reconstruction metrics
- module count
- module reuse
- comparison with one global model
- comparison with one-model-per-task solution


If the answer is No:

Do not proceed automatically.

Generate a report describing:

- why module sharing failed
- whether state representation is insufficient
- whether module granularity is inappropriate
- whether nonlinear local models are necessary
- whether the original research hypothesis should be revised

and wait for human review.


# 43. Current Implementation Priority

For now implement only:

1. repository skeleton
2. AGENTS.md
3. ExperimentSpec schema
4. experiment result storage
5. SCONE runner
6. trajectory parser
7. dataset schema
8. minimal visualization
9. tests
10. one reproducible walking run


After completing these tasks:

STOP.

Generate:

docs/status/milestone_1_report.md

including:

- what was implemented
- commands to reproduce
- tests
- unresolved SCONE API / file-format issues
- recommended Milestone 2 implementation

Do not proceed to Milestone 2 without explicit user approval.