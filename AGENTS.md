# Research Agent Rules

This repository investigates whether reusable predictive-control modules and
local plasticity can support continual motor learning without backpropagation.

## Source of truth and current scope

- Read `RESEARCH_SPEC.md` before research or implementation work.
- Read `readme.md`, `docs/research_plan.md`, and the relevant experiment spec.
- The current repository contains initial documentation, not a working simulator
  or experimental evidence. Do not describe planned components as implemented.
- Keep the research specification intact unless the user requests a change.
  Record material research decisions in `docs/decisions/`.
- Write project documentation in Japanese by default. Preserve technical names,
  identifiers, and the meaning of the research specification.

## Scientific priority

Do not optimize benchmark scores by silently changing the scientific question.
The priority is module reuse, composition, retention, novelty detection, and
adaptation under changing body dynamics and environments.

## Experimental discipline

Every experiment must:

1. reference an explicit hypothesis,
2. define one primary experimental change,
3. use fixed evaluation tasks,
4. use multiple random seeds,
5. retain failed experiments,
6. generate machine-readable metrics,
7. compare against an explicit baseline.

Experiments enter through YAML files in `experiments/specs/`.
Draft specifications containing unresolved required fields must not be executed.
Fix success criteria, seeds, data splits, metrics, and evaluation conditions
before execution. Keep pilot data separate from confirmatory evaluation data.

## Bio-MOSAIC constraint

No backpropagation or gradient-based synaptic weight update is allowed inside
experiments explicitly labeled Bio-MOSAIC. Gradient methods are allowed only
for explicit baseline models.

Do not attribute modularity benefits to non-BP learning without comparisons
that separate the two effects. SCONE optimization is an evolutionary search
proxy, not a faithful simulation of biological evolution.

## Implementation discipline

Before implementing a complex mechanism:

1. implement the smallest falsifiable version,
2. write tests,
3. run it on a minimal toy system,
4. only then integrate it with full SCONE.

Start discovery with local linear models and change point detection.
Do not start with neural networks. Treat single-trajectory discovery as a
preliminary check, not evidence of cross-condition reuse.

Do not pre-implement SNN, STDP, hierarchical module creation, or novelty-triggered
module growth before the required earlier evidence exists. Use anonymous module
IDs until their behavior has been analyzed; do not hard-code biological labels.

## Reporting and reproducibility

Never claim an experiment supports a hypothesis unless predefined success
criteria are satisfied. Negative results must be preserved. Do not delete
failed runs, selectively change seeds, or silently change benchmark conditions.
Do not change metrics after observing results without creating a new experiment
specification.

Generate `metrics.json` from code; never hand-write experimental values.
Capture the commit, resolved configuration, experiment spec, Python and
dependency versions, SCONE version, engine, OS, seeds, timestamp, and data
provenance for every run. Preserve stdout, stderr, and process exit status.
Distinguish genuine SCONE results from fixtures and mock runs.

Fit normalization and model selection on training/development data only.
Prevent overlap between training and evaluation episodes and conditions.
Do not use future observations to choose the current action.

## Research gates

Current implementation priority is `RESEARCH_SPEC.md` §32 and §43:
repository skeleton, ExperimentSpec validation, result storage, SCONE runner,
trajectory parser, dataset schema, minimal visualization, tests, and one
reproducible walking run.

After completing this work, STOP and write
`docs/status/milestone_1_report.md` with implementation status, reproduction
commands, tests, unresolved simulator issues, and the recommended next step.
Do not proceed to Milestone 2 without explicit user approval.

Before SNN or biological plasticity, report whether multiple SCONE locomotion
conditions compress into shared predictive-control modules, including held-out
conditions, prediction and control errors, module count, reuse, a global-model
baseline, and a one-model-per-task baseline.

If sharing fails, document the evidence and possible causes, then wait for human
review instead of proceeding automatically.
