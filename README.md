# Digital twin-based multi-objective optimization of adaptive abemaciclib dosing

Code for the paper

> Xu, Y.; Wu, C.; Qin, Y.; Li, F.; Mao, W.; Zhang, C.; Liu, S. Digital Twin-Based Multi-Objective
> Optimization of Adaptive Abemaciclib Dosing Policies in Hormone Receptor-Positive Advanced Breast
> Cancer.

The repository contains the digital twin, the four optimization problems, the 20 compared
algorithms (or the files needed to run them), and every script that produces the numbers, figures,
and tables of the paper.

## Contents

1. [Overview](#overview)
2. [Optimization problems](#optimization-problems)
3. [Algorithms](#algorithms)
4. [Repository layout](#repository-layout)
5. [Installation](#installation)
6. [Evaluating a dosing policy](#evaluating-a-dosing-policy)
7. [Reproducing the study](#reproducing-the-study)
8. [Figures and tables of the paper](#figures-and-tables-of-the-paper)
9. [Random seeds](#random-seeds)
10. [Third-party code](#third-party-code)
11. [License](#license)

## Overview

The digital twin simulates virtual patients treated with abemaciclib in a closed loop:

- a one-compartment pharmacokinetic model;
- the Friberg model of neutrophils;
- a tumor-size and progression-free survival model;
- a continuous-time Markov chain of diarrhea grades;
- cumulative hazards of other adverse reactions;
- the label dose-modification rules, which withhold, reduce, and discontinue the drug.

The twin is calibrated to published summary data of MONARCH 1, MONARCH 2, and the MONARCH 3 control
arm, and is kept as an ensemble of 20 parameter sets. It is then checked against settings that were
not calibration targets. No patient-level data are used: all inputs are published trial summaries
coded in `twin/trials.py` and `twin/params.py`.

A dosing policy adds rules to the label rules. Policies are optimized for three objectives under
three constraints, and 10 multi-objective evolutionary algorithms are compared with 10
multi-objective swarm intelligence algorithms.

The code uses four experiment labels:

| Label | Experiment |
|---|---|
| E1 | Calibration, numerical verification, and credibility checks of the twin |
| E2 | Comparison of the 20 algorithms on the instances P1-P4 |
| E3 | Analysis of the P1 policies on the test population |
| E4 | Scenario analysis on P1 |

## Optimization problems

All four instances are defined in `twin/problems.py`.

| Instance | Setting | Decision variables | Efficacy objective |
|---|---|---|---|
| P1 | First line, aromatase inhibitor partner (MONARCH 3 population) | 8 policy variables | Restricted mean survival time |
| P2 | Second line, fulvestrant partner (MONARCH 2 population) | 8 policy variables | Restricted mean survival time |
| P3 | As P1, fixed weekly schedule | 24 integers: dose level of weeks 1-24 (0 = drug holiday, 1-3 = 50/100/150 mg twice daily), then 150 mg | Restricted mean survival time |
| P4 | As P1, risk-averse | 8 policy variables | Mean progression-free survival of the worst 20% of patients |

The eight policy variables of P1, P2, and P4, in the order of the decision vector:

| # | Variable | Range | Meaning |
|---|---|---|---|
| 1 | `d0` | 1-3 (integer) | Starting dose level: 50, 100, or 150 mg twice daily |
| 2 | `d_tgt` | 2-3 (integer) | Target dose level: 100 or 150 mg twice daily |
| 3 | `tau_esc` | 7-28 days (integer) | Escalation interval |
| 4 | `theta_ANC` | 1.0-2.0 ×10⁹/L (real) | Minimum neutrophil count for escalation |
| 5 | `w_diar` | 0-14 days (integer) | Days without grade ≥ 2 diarrhea required before escalation |
| 6 | `n_pre` | 0-3 (integer) | Preventive reduction after this number of grade ≥ 2 diarrhea episodes in the first 8 weeks; 0 switches it off |
| 7 | `tau_re` | 28-84 days (integer) | Days without grade ≥ 2 toxicity and without a dose change before the dose ceiling is raised again |
| 8 | `r_max` | 0-2 (integer) | Maximum number of such re-escalations |

Objectives, all minimized:

| Objective | Definition |
|---|---|
| `f1` | Negative restricted mean progression-free survival time over 24 months, in months (P4: negative mean of the worst 20%) |
| `f2` | Probability of grade ≥ 3 neutropenia |
| `f3` | Mean number of days with grade ≥ 2 diarrhea in the first 6 months |

Constraints, feasible when ≤ 0. The label regimen (150 mg twice daily with the label rules) is
evaluated on the same patients and the same random numbers:

| Constraint | Definition |
|---|---|
| `g1` | 0.6 − median relative dose intensity |
| `g2` | Probability of grade 4 neutropenia − that of the label regimen |
| `g3` | Probability of discontinuation − that of the label regimen |

Each optimization run evaluates policies on a batch of 200 virtual patients, 10 from each of the 20
parameter sets. The final solutions of every run are re-evaluated on a test population of 10,000
patients, and the selected P1 policies again on an independent validation population of 10,000
patients.

## Algorithms

All algorithms use a population of 100 and 10,000 evaluations; the other settings are in the paper.

| Family | Algorithms | Implementation |
|---|---|---|
| Multi-objective evolutionary algorithms | SPEA2, NSGA-II, GDE3, MOEA/D-DE, NSGA-III, AGE-MOEA-II, LMPFE, CMOES, APSEA, CMOCEA-DDAP | PlatEMO v4.16 under GNU Octave (`platemo/`) |
| Multi-objective swarm intelligence algorithms | MOGWO, MSSA, MOAVOA, MOCGO, MOEOSMA, MOGNDO, MORIME, MaOMPA, MOSRS, C-MOBRO | Python ports of the authors' MATLAB code (`mosi/`) |

PlatEMO evaluates a whole population per call through two named pipes served by
`twin/platemo_server.py`, so both families use the same Python implementation of the problems.

Algorithms without constraint handling of their own receive Deb's constraint-domination rule
through the objectives: SPEA2, MOEA/D-DE, and LMPFE through `platemo/twin_constraints.m`, and the
swarm intelligence algorithms through `twin/si_runner.py`. The true objectives and constraints are
stored for every run.

## Repository layout

| Path | Content |
|---|---|
| `twin/params.py`, `twin/trials.py` | Model constants; trial settings and published targets |
| `twin/population.py` | Virtual patients and their random numbers |
| `twin/kernel.py` | Simulation kernel (Numba, 6-hour time grid) |
| `twin/problems.py`, `twin/scenarios.py` | Instances P1-P4; scenarios of E4 |
| `twin/cmaes.py` | CMA-ES used for the calibration |
| `twin/platemo_server.py`, `twin/true_values.py` | Evaluation server for PlatEMO; recomputation of true values after runs with Deb's rule |
| `twin/si_runner.py` | Runner of the swarm intelligence algorithms |
| `scripts/` | Calibration, ensemble selection, verification, sensitivity analysis; task lists and single runs |
| `scripts/analysis/` | Collection of runs, test-population evaluation, metrics and statistics, policy analysis, figures, LaTeX tables |
| `platemo/` | Problem classes `ABEMA_P1`-`ABEMA_P4`, headless runner `run_twin.m`, Octave support (`octave/`, `shims/`) |
| `mosi/codes/<ID>_<Name>/impl.py` | Python ports of the 10 swarm intelligence algorithms |
| `mosi/sicore/` | Evaluation-budget wrapper of the ports |
| `mosi/SOURCES.md` | Paper, repository, and commit of each original code |
| `jobs/` | PBS job scripts of the steps below |
| `tests/` | Unit tests |
| `calibration/`, `paper/figures/`, `paper/tables/` | Empty output folders |

No data and no results are distributed with the repository.

## Installation

Python 3.11 or later:

```bash
pip install -r requirements.txt
```

The 10 evolutionary algorithms also need:

- GNU Octave (tested with 10.3) with the Octave Forge `statistics` package (tested with 1.7.7);
- PlatEMO v4.16 (https://github.com/BIMK/PlatEMO, commit `d25e65d`). PlatEMO is not included. An
  Octave-compatible copy is made in `platemo/PlatEMO_oct` with

  ```bash
  bash platemo/octave/apply_patches.sh /path/to/PlatEMO/PlatEMO
  ```

  The original clone is not modified. Every patch is listed in `platemo/octave/PATCHES.md`.

Environment variables read by the scripts:

| Variable | Default | Meaning |
|---|---|---|
| `PYTHON` | `python` | Python interpreter |
| `OCTAVE_CLI` | `octave-cli` | Octave executable |
| `PLATEMO_OCT` | `platemo/PlatEMO_oct` | Patched PlatEMO copy |
| `OCTAVE_PKG_LIST` | not set | Local Octave package list |
| `MOSI_ROOT` | `mosi` | Folder of the swarm intelligence algorithms |
| `NPAR` | `32` | Parallel runs per job |
| `FIFO_ROOT` | `/dev/shm` | Folder of the named pipes |
| `TWIN_SCENARIO` | `base` | Scenario of E4 (`twin/scenarios.py`) |

Run the tests from the repository root:

```bash
python -m pytest -q tests
```

Tests that need the calibration files of step 1 below, or Octave, are skipped until these are
available.

## Evaluating a dosing policy

After the calibration (step 1 below), a policy is evaluated in a few lines. Run from the repository
root:

```python
import sys
sys.path.insert(0, "scripts")   # twin.problems imports the calibration module from scripts/

from twin.problems import Instance

inst = Instance("calibration/ensemble_time.json", "calibration/stage3_time.json",
                seed=3000001, name="P1")          # 200 virtual patients

label = [3, 3, 14, 1.0, 0, 0, 28, 0]      # 150 mg twice daily, label rules only
policy = [2, 3, 28, 1.59, 14, 1, 84, 2]   # start at 100 mg, escalate to 150 mg after 28 days

F, G = inst.evaluate([label, policy])
print(F)   # rows: policies; columns: f1, f2, f3
print(G)   # rows: policies; columns: g1, g2, g3 (feasible when all <= 0)
```

`seed` fixes the virtual patients and their random numbers, so two policies evaluated on the same
instance are compared on identical patients. The second policy above is the knee policy of the
P1 reference front reported in the paper.

## Reproducing the study

All commands are run from the repository root. The scripts in `jobs/` are written for a PBS
cluster and request one 32-core node each:

```bash
qsub -v NAME=value,... jobs/<script>.sh
```

Without PBS, set the same variables and run the script with `bash`. Finished runs are skipped, so
interrupted jobs can be submitted again.

### Step 1. Calibration and credibility checks (E1)

Run every line for `STRUCTURE=time` (main analysis, time-based diarrhea tolerance) and for
`STRUCTURE=exposure` (cumulative-exposure tolerance), in this order:

```bash
qsub -v STEP=stage12,STRUCTURE=time   jobs/calibrate.sh   # neutrophils, diarrhea, rule adherence
qsub -v STEP=stage3,STRUCTURE=time    jobs/calibrate.sh   # first-line efficacy
qsub -v STEP=ensemble,STRUCTURE=time  jobs/calibrate.sh   # 20 parameter sets
qsub -v STEP=submodels,STRUCTURE=time jobs/calibrate.sh   # reproduction of published sub-model results
qsub -v STEP=sobol,STRUCTURE=time     jobs/calibrate.sh   # global sensitivity analysis
qsub -v STEP=verify                   jobs/calibrate.sh   # numerical verification of the kernel
```

Output: `calibration/stage12_<structure>.json`, `stage3_<structure>.json`,
`ensemble_<structure>.json`, `check_submodels_<structure>.json`, `sobol_<structure>.json`, and
`verify_numerics.json`. All later steps read `ensemble_<structure>.json` and
`stage3_<structure>.json`.

### Step 2. Algorithm comparison (E2)

20 algorithms × 4 instances × 30 runs = 2400 runs, one per line of the task list:

```bash
python scripts/make_tasks.py --out results/tasks.txt
qsub -v TASKS=results/tasks.txt,FIRST=1,LAST=64 jobs/run_tasks.sh    # repeat up to line 2400
```

A single run can be started directly:

```bash
bash scripts/run_one.sh P1 moea NSGAII 3000001
bash scripts/run_one.sh P1 mosi M001_MOGWO 3000001
```

Output: `results/time/<instance>/<algorithm>/seed<seed>.mat` for the evolutionary algorithms and
`.npz` for the swarm intelligence algorithms, each with the final solutions and 20 checkpoints.

### Step 3. Analysis of E2 and E3

```bash
qsub -v STAGE=pre,RES=results,OUT=analysis jobs/analysis.sh                    # cross-check, collect
qsub -v OUT=analysis,INST=P1,NCHUNKS=8,C0=0,C1=7 jobs/test_eval.sh             # repeat for P2, P3, P4
qsub -v STAGE=post,OUT=analysis,NCHUNKS=8,FIG=paper/figures jobs/analysis.sh   # metrics, E3, figures
python scripts/analysis/e3_robust.py
qsub jobs/e3_members.sh
qsub jobs/validation.sh
python scripts/analysis/tables_tex.py
```

| Stage | Script | Output |
|---|---|---|
| Cross-check | `scripts/crosscheck_runs.py` | Stored solutions re-evaluated directly in Python, `analysis/crosscheck.txt` |
| Collect | `scripts/analysis/collect.py` | `analysis/runs_<instance>.pkl` |
| Test population | `scripts/analysis/test_eval.py` | `analysis/test_<instance>.pkl` |
| Metrics and statistics | `scripts/analysis/metrics.py` | `analysis/metrics_<instance>.pkl`, `analysis/summary.json`, `analysis/tables/*.csv` |
| Policy analysis | `scripts/analysis/e3.py`, `e3_robust.py` | `analysis/e3_P1.pkl`, `analysis/e3_robust_P1.json` |
| Parameter uncertainty | `scripts/analysis/e3_members.py` | `analysis/e3_members_P1.json` |
| Validation population | `scripts/analysis/validation.py` | `analysis/validation_P1.json` |

The metrics are the hypervolume and IGD⁺ on the test population. Every algorithm is compared with
the best one by the Wilcoxon signed-rank test paired by seed, with the Holm correction.

### Step 4. Scenario analysis (E4)

The three algorithms with the highest mean hypervolume on P1 are run 10 times in each of 15
scenarios (450 runs): the 3 × 3 grid of drug efficacy and diarrhea tolerance without its center,
neutrophil sensitivity, the dose-intensity bound, the time window, and the cumulative-exposure
tolerance.

```bash
bash scripts/prepare_e4.sh CMOCEADDAP CMOES APSEA
qsub -v TASKS=results_e4/tasks.txt,FIRST=1,LAST=64 jobs/run_e4_tasks.sh    # repeat up to line 450
qsub -v SCENS="base exposure eff_low_tol_low eff_low_tol_mid eff_low_tol_high eff_mid_tol_low eff_mid_tol_high eff_high_tol_low eff_high_tol_mid eff_high_tol_high anc_low anc_high rdi_0.5 rdi_0.7 window_18 window_36" jobs/e4_analysis.sh
python scripts/analysis/e4_summary.py --e4 analysis_e4 --out paper
python scripts/analysis/table_e4_tex.py
```

`prepare_e4.sh` links the first 10 runs of the main experiment as the `base` scenario, so every
scenario is compared with the main experiment on the same patients and random numbers.

Output: `results_e4/<scenario>/`, `analysis_e4/<scenario>/`, and `paper/tables/e4_summary.json`.

### Step 5. Remaining figures and tables

```bash
python scripts/analysis/fig_twin.py paper/figures
qsub jobs/extra_results.sh
python scripts/analysis/fig_members.py
```

## Figures and tables of the paper

Figures are written to `paper/figures` as PDF and PNG, and LaTeX tables to `paper/tables`.

| Item | File | Script |
|---|---|---|
| Figure 1, study overview | `fig_twin` | `scripts/analysis/fig_twin.py` |
| Figure 2, validation of the twin | `fig_validation` | `scripts/analysis/figures.py` |
| Figure 3, Sobol indices | `fig_sobol` | `scripts/analysis/extra_results.py` |
| Figure 4, test hypervolume of the runs | `fig_hv_box` | `scripts/analysis/figures.py` |
| Figure 5, ranks of the algorithms | `fig_ranks` | `scripts/analysis/extra_results.py` |
| Figure 6, optimization batch versus test population | `fig_generalization` | `scripts/analysis/extra_results.py` |
| Figure 7, reference front of P1 | `fig_front_P1` | `scripts/analysis/figures.py` |
| Figure 8, population curves of representative policies | `fig_trajectories_P1` | `scripts/analysis/figures.py` |
| Figure 9, decision variables on the reference front | `fig_policy_vars` | `scripts/analysis/extra_results.py` |
| Figure 10, differences within the 20 parameter sets | `fig_members` | `scripts/analysis/fig_members.py` |
| Figure 11, reference fronts of P2, P3, P4 | `fig_instances` | `scripts/analysis/extra_results.py` |
| Figure 12, weekly schedules of P3 | `fig_p3_schedules` | `scripts/analysis/extra_results.py` |
| Figure 13, knee policy in the scenarios | `fig_e4_knee` | `scripts/analysis/e4_summary.py` |
| Table 4, test hypervolume | `table_hv.tex` | `scripts/analysis/tables_tex.py` |
| Table 5, statistics of the algorithms on P1 | `table_p1_algs.tex` | `scripts/analysis/extra_results.py` |
| Table 6, representative P1 policies | `table_reps.tex` | `scripts/analysis/extra_results.py` |
| Table 7, policies on the four instances | `table_instances.tex` | `scripts/analysis/extra_results.py` |
| Table S2, IGD⁺ | `table_igd.tex` | `scripts/analysis/tables_tex.py` |
| Table S3, scenario analysis | `table_e4.tex` | `scripts/analysis/table_e4_tex.py` |

`scripts/analysis/extra_results.py` also writes `analysis/extra_results.json`, which holds the
numbers quoted in the text. `scripts/analysis/figures.py` additionally writes `fig_convergence`,
which is not shown in the paper.

## Random seeds

| Use | Seeds |
|---|---|
| Optimization runs of E2, run k = 1-30 | 3,000,001-3,000,030 |
| Optimization runs of E4, run k = 1-10 | 3,000,001-3,000,010 |
| Test population | 9,000,001 |
| Validation population | 9,000,002 |

The seed of run k fixes both the random numbers of the algorithm and the patient batch of the run,
so all algorithms meet the same patients in run k. Patients are simulated in parallel, and the
results do not depend on the number of threads.

## Third-party code

- The swarm intelligence algorithms in `mosi/codes/` are Python ports of the MATLAB code published
  by the authors of each algorithm. `mosi/SOURCES.md` gives the paper, the repository, and the
  commit of each. The docstring of each `impl.py` lists the differences from the original code, and
  the license file of the original repository is kept where one exists.
- PlatEMO is developed by the BIMK group and is not redistributed here.
- `platemo/shims/octave_hist_orig.m` is part of GNU Octave.

## License

MIT License for the code of this repository (see `LICENSE`). Third-party code keeps its own
license.
