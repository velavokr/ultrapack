---
name: ml-experiments
description: Use before launching any training run, hyperparameter sweep, or model evaluation. Enforces pre-compute checks, leak prevention, reproducibility, monitoring live at launch, scale-up gating. Auto-triggers on train.py, training configs, model files.
---

# ML Experiments

Compute is expensive. Bugs in ML code fail silently — loss still goes down, just on the wrong thing. Every check in this skill exists because skipping it has burned someone.

## Iron law — never launch a full run without passing every pre-compute check

Silent failure is the default in ML; you have to work to make it loud.

## Pre-compute check 1 — overfit one batch, prove the model can memorize

Load a single batch. Train on it alone. Loss must go to near-zero within a few hundred steps.

If it doesn't: the model can't even memorize the data. The bug is in the model, the loss, or the data pipeline — not the hyperparameters. Fix before anything else.

## Pre-compute check 2 — small-run dynamics sanity

Before the full run: short run (e.g. 100 steps, 1% of data). Check:

- Loss decreasing (not NaN, not exploding, not flat)
- Gradients finite
- Learning rate schedule behaving as expected
- GPU utilization sane (not 5%)

## Pre-compute check 3 — data loader shape / dtype / stats

Print shape, dtype, min/max/mean of one batch of each split. Verify:

- Shapes match what the model expects
- Dtypes are what you intended (fp32 vs bf16 vs int)
- Input ranges are reasonable (normalized? clipped?)
- Labels map correctly to classes (visualize a few)

## Leak prevention — splits, targets, time, preprocessor

Data leaks make numbers look good and models ship broken.

- Split isolation: train/val/test are disjoint at the entity level (by user, by document, by session), not just by row
- No target leakage: no feature derived from the label, no feature available only at training time
- No temporal leakage: for time-series, train strictly precedes val strictly precedes test
- Preprocessor fit on train only: scalers, vocabularies, imputers `.fit()` on train, `.transform()` on val/test
- No val/test peeking: don't look at test metrics until the experiment is over

## Reproducibility — every run must be re-runnable

Record:

- Seed (for `random`, `numpy`, framework RNG, dataloader workers)
- Full config (dumped to the run directory, not just passed via CLI)
- Run ID (timestamped, unique, logged at launch)
- Deterministic checkpoint paths (same run → same paths, no overwrites across runs)
- Git SHA of the code that produced the run

If you can't re-run it, you can't debug it.

## Monitoring — dashboards streaming before the launch command runs

Live dashboards (wandb / tensorboard / mlflow) must be up before you hit Enter. Not "I'll check logs later". Watch the first few minutes:

- Is loss moving?
- Are eval metrics logged?
- Are the curves shaped like previous working runs?

Define the eval metric up front. Decide the decision rule before the run ("I will keep the checkpoint with highest val F1") — not after seeing numbers.

## Scale-up gating — tiny → small → medium → full

Never jump straight to full scale. Each gate is a go/no-go.

<gates>
1. Tiny — overfit one batch, single step verified
2. Small — 1% data, short run, sanity check curves
3. Medium — 10% data or 10% steps, full eval pipeline exercised
4. Full — only after every prior gate passed
</gates>

If small-run metrics look wrong, fix before burning compute on medium.

## Experiment hygiene — one change at a time, baseline locked, failed runs kept

- One change at a time. Comparing two experiments that differ in two things teaches you nothing.
- Baseline locked. Don't change the baseline between experiments.
- Never peek at test mid-experiment. If you do, that split is now contaminated — treat it as val.
- Keep failed runs. Negative results are data. Log them, don't delete.

## Suspect leakage when numbers look too good

ML models don't usually jump 20 points. If metrics suddenly soar:

- Re-check split isolation
- Re-check preprocessor fit scope
- Re-check feature list for label-derived features
- Evaluate on an obviously-held-out test set you just constructed

## Red flags — STOP

<system-reminder>
- Loss goes to zero too fast → leakage or trivial task
- Val metric >> train metric → split contamination or eval bug
- NaN loss → bad init, bad data, bad learning rate; don't "just add gradient clipping and retry"
- GPU at 5% utilization → data loader bottleneck; full run wastes 20× the compute
- "I'll fix monitoring after launch" → no, you won't
</system-reminder>

## Terminal state

All pre-compute checks passed → launch full run with monitoring live. Document the run ID and decision rule in the task file or experiment log before checkpoint 0.
