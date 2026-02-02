# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is an **environment calibration workflow** for malaria modeling using EMOD (Epidemiological MODeling) simulations. The codebase uses **Bayesian optimization with Gaussian Process emulators** to calibrate environmental parameters (temperature, habitat multipliers) to match reference epidemiological data (prevalence, incidence, EIR) for specific field sites.

## Key Commands

### Running Calibration

```bash
# Submit calibration job to SLURM cluster
sbatch simulations/sbatch_run_calib.sh

# Note: Update VENV_PATH in both manifest.py and sbatch_run_calib.sh before running
```

### Python Environment Setup

See detailed instructions in README.md Step 1. Key dependencies:
- Python 3.9
- PyTorch 1.11 with CUDA 11.2
- emodpy-malaria (3.4 or 4.1.0 depending on setup)
- botorch 0.8.1
- gpytorch
- idmtools-platform-slurm 1.7.11

Install the two internal packages in editable mode:
```bash
cd calibration_common
pip install -e . --index-url=https://packages.idmod.org/api/pypi/pypi-production/simple

cd ../environment_calibration_common
pip install -e .
```

### Analysis

Post-calibration analysis is automatically run after the BO loop completes. For additional plots:
```bash
# R Markdown for convergence plots (requires updating exp_label inside)
# Run in RStudio or via Rscript
Rscript -e "rmarkdown::render('simulations/post_calibration_plots.Rmd')"
```

## Architecture

### Three-Component System

1. **calibration_common/** (Git submodule)
   - Generic Bayesian optimization infrastructure
   - `bo.py`: Core BO workflow (checkpointing, initialization, iteration control)
   - `batch_generators/`: Acquisition functions (TurboThompsonSampling is primary)
   - `emulators/GP.py`: Gaussian Process models (ExactGP is primary)
   - `post_calibration_GP.py`: Post-hoc analysis including length scale analysis

2. **environment_calibration_common/** (Git submodule)
   - EMOD-specific simulation and analysis logic
   - `my_func.py`: Orchestrates parameter translation → simulation → analysis → scoring
   - `run_sims.py`: Creates and submits EMOD simulations via idmtools
   - `helpers.py`: Simulation configuration (demographics, climate, interventions, reports)
   - `analyzers/`: Extract simulation outputs (InsetChart, MalariaSummaryReport)
   - `compare_to_data/`: Score simulations against reference datasets
   - `translate_parameters.py`: Converts unit space [0,1] to EMOD parameter values

3. **Top-level project directory**
   - `simulations/run_calib.py`: Main entry point defining the Problem class
   - `simulations/manifest.py`: User-configurable paths and experiment settings
   - `simulation_inputs/`: Site-specific configuration CSVs and climate/demographics files
   - `reference_datasets/`: Target data for calibration (prevalence, incidence)
   - `simulations/output/<exp_label>/`: Results organized by BO round (LF_0, LF_1, ...)

### Workflow Execution Flow

1. **run_calib.py** defines `Problem` class with `__call__(X)` method
2. **BO.initRandom()** generates initial Sobol samples, calls `Problem(X)`
3. **Problem.__call__()** invokes `myFunc(X)` from environment_calibration_common
4. **myFunc()** translates parameters → submits simulations → waits for completion → runs analyzers → computes scores
5. After each batch, **BO** fits GP emulator and uses **TurboThompsonSampling** to suggest next batch
6. Trust region expands/contracts based on success/failure counters
7. Loop continues until `max_eval` simulations completed
8. **post_calibration_analysis()** runs final GP fits and generates diagnostic plots

### Parameter Space Translation

Defined in `simulations/parameter_key.csv`:
- **Temperature_Shift**: Linear scaling from unit [0,1] to [min, max]
- **Habitat Multipliers** (CONST, TEMPR, WATEV): Log-scale transformation

### Scoring System

Defined in `simulation_inputs/weights.csv` and `simulation_coordinator.csv`:
- **eir_score**: Penalizes unrealistic EIR (monthly EIR >= 100 or == 0)
- **shape_score**: Compares normalized monthly incidence patterns (via log-likelihood)
- **intensity_score**: Compares average annual incidence (exponential of relative error)
- **prevalence_score**: Compares PCR or microscopy prevalence (RMSE across month-years)

Final score = -(weighted sum). Negative because BO maximizes. Missing scores default to penalty of 10.

### Key Configuration Files

- `simulation_inputs/simulation_coordinator.csv`: Site metadata, reference datasets, which objectives to use
- `simulation_inputs/calibration_coordinator.csv`: BO hyperparameters (init_size, batch_size, max_eval, failure_limit)
- `simulations/parameter_key.csv`: Parameter bounds and transformations
- `simulation_inputs/weights.csv`: Objective function weights

### Output Structure

`simulations/output/<exp_label>/`:
- `LF_<n>/`: Each BO round
  - `translated_params.csv`: Parameter sets evaluated this round
  - `emod.best.csv`, `emod.ymax.txt`: Best parameter set and score (only when improved)
  - `EIR_range.csv`, `ACI.csv`: Summary statistics
  - `incidence_<site>.png`, `prevalence_<site>.png`: Comparison plots
  - `SO/<site>/`: Full simulation outputs (InsetChart.csv, etc.)
- `all_LL.csv`: All scores across all rounds
- `performance/GP/`: Post-calibration length scale analysis

## Development Notes

### Adding New Objectives

Follow the workflow described in README.md "Tips for adding new objectives":
1. Add flags/paths to `simulation_inputs/simulation_coordinator.csv`
2. Add weight to `simulation_inputs/weights.csv`
3. Modify `environment_calibration_common/helpers.py` to add reports
4. Create analyzer in `environment_calibration_common/analyzers/analyzer_collection.py`
5. Add scoring logic in `environment_calibration_common/compare_to_data/calculate_all_scores.py`
6. Wire up in `environment_calibration_common/compare_to_data/run_full_comparison.py`

### Platform-Specific Configuration

This codebase is designed for Northwestern's Quest HPC cluster with SLURM:
- `manifest.py` defines platform as "CALCULON" or "SLURM_LOCAL"
- `SIF_PATH` points to Singularity image for EMOD execution
- `utils_slurm.py` handles chain job submission
- For local/COMPS execution, modify platform settings in `manifest.py`

### Key Variables to Update Before Running

In `simulations/manifest.py`:
- `SITE`: Name of calibration site (must match simulation_coordinator.csv)
- `EXPERIMENT_LABEL`: Unique identifier for this calibration run
- `VENV_PATH`: Path to Python virtual environment (commented out, but referenced in sbatch script)

In `simulations/sbatch_run_calib.sh`:
- Python path: Update to your virtual environment

## Git Submodules

This repository includes two submodules:
```bash
# Clone with submodules
git clone <repo-url> --recursive

# Update submodules
git submodule update --init --recursive

# Pull latest changes in submodules
git submodule update --remote
```

**calibration_common** and **environment_calibration_common** are maintained in separate repositories. Changes to core BO or EMOD logic should be made in those repos.
