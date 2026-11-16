# Digital twin-based multi-objective optimization of adaptive abemaciclib dosing

Code for the paper "Digital Twin-Based Multi-Objective Optimization of Adaptive Abemaciclib Dosing
Policies in Hormone Receptor-Positive Advanced Breast Cancer".

The digital twin simulates virtual patients treated with abemaciclib: a one-compartment
pharmacokinetic model, the Friberg model of neutrophils, a tumor-size and progression-free survival
model, a continuous-time Markov chain of diarrhea grades, cumulative hazards of other adverse
reactions, and the label dose-modification rules in a closed loop. The twin is calibrated to
published summary data of MONARCH 1, MONARCH 2 and the MONARCH 3 control arm and kept as an ensemble
of 20 parameter sets. Adaptive dosing policies are optimized for restricted mean progression-free
survival, grade 3 or worse neutropenia and days with grade 2 or worse diarrhea, under constraints
relative to the label regimen, with 10 multi-objective evolutionary algorithms (PlatEMO) and 10
multi-objective swarm intelligence algorithms (Python ports).

## Repository layout

| Path | Content |
|---|---|
| `twin/` | Model constants (`params.py`), virtual patients (`population.py`), simulation kernel (`kernel.py`, Numba), trial settings and targets (`trials.py`), problem instances P1-P4 (`problems.py`), scenarios (`scenarios.py`), CMA-ES (`cmaes.py`), runners for PlatEMO (`platemo_server.py`, `true_values.py`) and for the swarm intelligence algorithms (`si_runner.py`) |
| `scripts/` | Calibration, ensemble selection, verification and sensitivity analysis; task lists and single runs |
| `scripts/analysis/` | Collection of runs, test-population evaluation, metrics and statistics, policy analysis, figures and LaTeX tables |
| `platemo/` | PlatEMO problem classes `ABEMA_P1`-`ABEMA_P4`, the headless runner `run_twin.m`, and Octave support (`octave/`, `shims/`) |
| `mosi/` | Python ports of the 10 swarm intelligence algorithms (`codes/<ID>_<Name>/impl.py`, with the source and license of the original code) and the evaluation-budget wrapper `sicore/` |
| `jobs/` | PBS job scripts for the steps below |
| `tests/` | Unit tests |
| `calibration/` | Output folder of the calibration (step 1 below) |

Experiment labels used in the code: E1 calibration and credibility checks, E2 algorithm comparison
on P1-P4, E3 analysis of the P1 policies, E4 scenario analysis.

## Requirements

- Python 3.11 or later with the packages in `requirements.txt` (`pip install -r requirements.txt`).
- For the 10 evolutionary algorithms: GNU Octave (tested with 10.3) with the Octave Forge
  `statistics` package (1.7.7), and PlatEMO v4.16 (https://github.com/BIMK/PlatEMO, commit
  `d25e65d`). PlatEMO is not included; make the Octave-compatible copy with

  ```
  bash platemo/octave/apply_patches.sh /path/to/PlatEMO/PlatEMO
  ```

  The patches are listed in `platemo/octave/PATCHES.md`.

Environment variables: `PYTHON` (Python interpreter, default `python`), `OCTAVE_CLI` (default
`octave-cli`), `PLATEMO_OCT` (patched PlatEMO copy, default `platemo/PlatEMO_oct`), `OCTAVE_PKG_LIST`
(local Octave package list, optional), `MOSI_ROOT` (default `mosi`), `NPAR` (parallel runs per job,
default 32).

Run the tests from the repository root:

```
python -m pytest -q tests
```

The tests of the problem instances need the calibration files of step 1 and are skipped until
they exist.

## Reproducing the study

All commands are run from the repository root. The job scripts in `jobs/` are written for PBS
(`qsub -v NAME=value,... jobs/<script>.sh`); the same steps can be run with `bash` after setting the
variables.

1. Calibration and checks (E1), for `STRUCTURE=time` and `STRUCTURE=exposure`:

   ```
   qsub -v STEP=stage12,STRUCTURE=time jobs/calibrate.sh
   qsub -v STEP=stage3,STRUCTURE=time jobs/calibrate.sh
   qsub -v STEP=ensemble,STRUCTURE=time jobs/calibrate.sh
   qsub -v STEP=submodels,STRUCTURE=time jobs/calibrate.sh
   qsub -v STEP=sobol,STRUCTURE=time jobs/calibrate.sh
   qsub -v STEP=verify jobs/calibrate.sh
   ```

   The later steps read `calibration/ensemble_<structure>.json` and
   `calibration/stage3_<structure>.json`. No data or results are distributed with this repository;
   all inputs of the calibration are the published trial summaries coded in `twin/trials.py` and
   `twin/params.py`.

2. Algorithm comparison (E2): 20 algorithms, 4 instances, 30 runs each, population 100,
   10,000 evaluations, seeds 3,000,001-3,000,030.

   ```
   python scripts/make_tasks.py --out results/tasks.txt
   qsub -v TASKS=results/tasks.txt,FIRST=1,LAST=64 jobs/run_tasks.sh     # repeat for all lines
   ```

3. Analysis of E2 and E3:

   ```
   qsub -v STAGE=pre,RES=results,OUT=analysis jobs/analysis.sh
   qsub -v OUT=analysis,INST=P1,NCHUNKS=8,C0=0,C1=7 jobs/test_eval.sh   # for P1-P4
   qsub -v STAGE=post,OUT=analysis,NCHUNKS=8,FIG=paper/figures jobs/analysis.sh
   python scripts/analysis/e3_robust.py
   qsub jobs/e3_members.sh
   qsub jobs/validation.sh        # independent validation population (seed 9,000,002)
   python scripts/analysis/tables_tex.py
   ```

4. Scenario analysis (E4) with the three algorithms with the highest mean hypervolume on P1:

   ```
   bash scripts/prepare_e4.sh CMOCEADDAP CMOES APSEA
   qsub -v TASKS=results_e4/tasks.txt,FIRST=1,LAST=64 jobs/run_e4_tasks.sh    # repeat for all lines
   qsub -v SCENS="exposure eff_low_tol_low ..." jobs/e4_analysis.sh
   python scripts/analysis/e4_summary.py --e4 analysis_e4 --out paper     # writes paper/tables/e4_summary.json
   python scripts/analysis/table_e4_tex.py
   ```

5. Remaining figures and tables:

   ```
   python scripts/analysis/fig_twin.py paper/figures
   qsub jobs/extra_results.sh
   python scripts/analysis/fig_members.py
   ```

Figures are written to `paper/figures` and LaTeX tables to `paper/tables`.

## Third-party code

The swarm intelligence algorithms in `mosi/codes/` are Python ports of the MATLAB code published by
the authors of each algorithm; `mosi/SOURCES.md` gives the paper, the repository and the commit of
each, and the docstring of each `impl.py` documents the differences from the original code. PlatEMO is developed by the BIMK
group and is not redistributed here. `platemo/shims/octave_hist_orig.m` is part of GNU Octave.

## License

MIT License for the code of this repository (see `LICENSE`); third-party code keeps its own license.
