# 01 Mandatory Entry Points

## What Counts As A Real Entry Point

For this repository, a “real entry point” is a user-invoked command or notebook that actually drives one stage of the paper workflow.

## Primary Entry Points

| Stage | User-facing command | Actual entry file | Paper alignment | Main outputs |
| --- | --- | --- | --- | --- |
| Algorithm eval / trace export | `cd algorithm && python -m run_eval ...` | `algorithm/run_eval.py` | `doc/paper/tex/5.semantic.tex`, `doc/paper/tex/6.vector.tex`, `doc/paper/tex/7.evaluation.tex` | traces, metadata, accuracy CSVs |
| Simulator | `cd simulator && python main.py ...` | `simulator/main.py` | `doc/paper/tex/7.evaluation.tex` | latency/energy/memory CSVs |
| Area/power summary | `cd simulator && python arch/accelerator.py --output_dir ...` | `simulator/arch/accelerator.py` | `doc/paper/tex/7.evaluation.tex` | `accelerator_area_power_buffer.csv` |
| Worst-case analysis | `cd simulator && python utils/analysis.py --trace_dir ...` | `simulator/utils/analysis.py` | `doc/paper/tex/7.evaluation.tex` worst/best-case analysis | `figure_13.svg` |
| Plotting | run notebooks in `evaluation_scripts/plot_scripts/ipynb_src/` | notebook files themselves | `doc/paper/tex/7.evaluation.tex` | figures/tables |

## The True Algorithm Entry

### `algorithm/run_eval.py`

- **Direct observation**: every algorithm shell wrapper eventually calls `python -m run_eval`.
- **Direct observation**: `algorithm/run_eval.py` owns:
  - Focus / CMC / Adaptiv / FrameFusion flags
  - trace export flags
  - quantization flags
  - accuracy CSV writing
- **Direct observation**: it enters `algorithm/lmms-eval/lmms_eval/evaluator.py`.

### Why It Matters

- It is the actual control plane for the paper’s algorithm experiments.
- It is where paper-facing outputs such as `accuracy.csv` and DSE accuracy tables are written.

## Algorithm Wrapper Scripts

| Script | Underlying entry | Paper role |
| --- | --- | --- |
| `algorithm/run_focus.sh` | `algorithm/run_eval.py` | Main Focus video experiments; trace generation and full accuracy |
| `algorithm/run_original.sh` | `algorithm/run_eval.py` | Dense baseline accuracy |
| `algorithm/run_framefusion.sh` | `algorithm/run_eval.py` | FrameFusion baseline accuracy |
| `algorithm/run_adaptiv.sh` | `algorithm/run_eval.py` | Adaptiv sparsity or accuracy |
| `algorithm/run_cmc.sh` | `algorithm/run_eval.py` | CMC sparsity or accuracy |
| `algorithm/run_dse.sh` | `algorithm/run_eval.py` | Focus DSE traces and rough DSE accuracy |
| `algorithm/run_focus_image.sh` | `algorithm/run_eval.py` | Image-VLM Focus experiments from the paper discussion section |
| `algorithm/run_adaptiv_image.sh` | `algorithm/run_eval.py` | Image-VLM Adaptiv baseline |
| `algorithm/run_original_image.sh` | `algorithm/run_eval.py` | Image-VLM dense baseline |

## The True Simulator Entry

### `simulator/main.py`

- **Direct observation**: all simulator shell wrappers eventually call `python main.py`.
- **Direct observation**: `simulator/main.py` owns:
  - main comparison runs
  - DSE modes
  - INT8 simulation
  - SEC-only ablation
  - image-task runs

### Simulator Wrapper Scripts

| Script | Underlying entry | Paper role |
| --- | --- | --- |
| `simulator/run_main_sim.sh` | `simulator/main.py` | Main performance and energy results |
| `simulator/run_dse_sim.sh` | `simulator/main.py` | DSE figure inputs |
| `simulator/run_image_sim.sh` | `simulator/main.py` | Image-VLM discussion table inputs |

## Important Auxiliary Entry Boundaries

These are not user-facing commands, but they are real internal entry boundaries.

| File | Why it matters |
| --- | --- |
| `algorithm/lmms-eval/lmms_eval/evaluator.py` | This is where `apply_focus(...)` and `apply_framefusion(...)` are triggered before inference. |
| `algorithm/focus/interface.py` | This is where supported models are detected and forward functions are replaced. |
| `simulator/core/simulator.py` | This is where Focus traces become latency, stall, DRAM, and energy results. |

## Notebook Entry Points

| Notebook | Paper artifact |
| --- | --- |
| `figure_9.ipynb` | speedup + energy + area/power breakdown |
| `figure_10.ipynb` | DSE |
| `figure_11.ipynb` | ablation |
| `figure_12.ipynb` | DRAM / activation analysis |
| `table_2.ipynb` | accuracy + computation sparsity |
| `table_4.ipynb` | INT8 study |
| `table_5.ipynb` | image-VLM study |

## Primary vs Auxiliary

### Primary

- `algorithm/run_eval.py`
- `simulator/main.py`
- `simulator/arch/accelerator.py`
- `simulator/utils/analysis.py`
- `evaluation_scripts/plot_scripts/ipynb_src/*.ipynb`

### Auxiliary

- `algorithm/run_*.sh`
- `simulator/run_*.sh`
- `algorithm/lmms-eval/lmms_eval/evaluator.py`
- `algorithm/focus/interface.py`
- `simulator/core/simulator.py`

## Bottom Line

- **Direct observation**: the repository has two truly central runnable programs: `algorithm/run_eval.py` and `simulator/main.py`.
- **Reasoned inference**: everything else is either a wrapper around them, an internal dispatch boundary, or a downstream reporting layer.
